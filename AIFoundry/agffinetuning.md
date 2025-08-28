# AGF Azure GenAI Foundation - Finetuning to create workflows

## Introduction

- Fine tune a model to build AGF workflows using API's
- These are consolidated API Microservices to use.
- So take the API definitions and create a data set for chat interactions.
- We will write a data set distillation code as well.

## Pre-requisites

- Familiarity with Azure AI Foundry
- Basic understanding of API development and microservices
- Access to the necessary Azure resources and permissions
- Using the GPT 4.1 nano from Azure AI Foundry
- using fine tuning as a service

## Implementation Steps

### Create distillation data set

- we need the input JSON with all API definitions and their descriptions.
- We are using Postman collections which has all the api
- It also has parameters and configurations for each API endpoint.
- Store in a folder
- Here is the code to create data set


```
#!/usr/bin/env python3
"""
generate_task_execution_dataset.py

Creates a synthetic task-execution dataset from scratch using the OpenAI Responses API (Azure OpenAI compatible).
Given an API catalog JSON describing available APIs, it produces a JSONL dataset with
(task, plan of API calls, validation checks, negatives, metadata) 500 samples by default.

Usage:
  python generate_task_execution_dataset.py \
      --api-catalog path/to/catalog.json \
      --output dataset.jsonl \
      --samples 500 \
    --model <azure-deployment-name> \
    [--seed-task "Buy two ground-shipping widgets under $50"]

Environment:
    For Azure OpenAI:
        - AZURE_OPENAI_ENDPOINT
        - AZURE_OPENAI_API_KEY (or AZURE_OPENAI_KEY)
        - (optional) AZURE_OPENAI_API_VERSION

Note: --model should be the Azure deployment name (not the base model name). This script targets the
{AZURE_OPENAI_ENDPOINT}/openai/vi/ endpoint and uses chat completions.

Docs referenced:
- Responses API & Python SDK: https://platform.openai.com/docs/api-reference/responses
- Structured Outputs (JSON schema): https://platform.openai.com/docs/guides/structured-outputs
- Rate limits & recommended backoff: https://platform.openai.com/docs/guides/rate-limits
"""

import argparse
import json
import os
import random
import threading
from concurrent.futures import ThreadPoolExecutor, as_completed
import time
import uuid
from pathlib import Path
from typing import Any, Dict, List, Optional, Tuple

from dotenv import load_dotenv
from jsonschema import ValidationError, validate
from openai import (
    APIConnectionError,
    APIError,
    APITimeoutError,
    AzureOpenAI,
    RateLimitError,
)
from tenacity import retry, retry_if_exception_type, stop_after_attempt, wait_random_exponential
from tqdm import tqdm

load_dotenv()


def load_json(path: str) -> Dict[str, Any]:
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)


def write_jsonl(records: List[Dict[str, Any]], path: str) -> None:
    with open(path, "w", encoding="utf-8") as f:
        for r in records:
            f.write(json.dumps(r, ensure_ascii=False) + "\n")


def build_dataset_schema() -> Dict[str, Any]:
    """Return the JSON Schema that model outputs must follow."""
    return {
        "type": "object",
        "additionalProperties": False,
        "required": ["task_id", "task_nl", "plan", "metadata"],
        "properties": {
            "task_id": {"type": "string"},
            "task_nl": {"type": "string"},
            "assumptions": {"type": "array", "items": {"type": "string"}},
            "plan": {
                "type": "object",
                "additionalProperties": False,
                "required": ["steps", "expected_end_state"],
                "properties": {
                    "global_constraints": {"type": "object"},
                    "steps": {
                        "type": "array",
                        "minItems": 1,
                        "items": {
                            "type": "object",
                            "additionalProperties": False,
                            "required": ["index", "action", "api", "arguments"],
                            "properties": {
                                "index": {"type": "integer", "minimum": 1},
                                "action": {"type": "string"},
                                "api": {"type": "string"},
                                "arguments": {"type": "object"},
                                "depends_on": {
                                    "type": "array",
                                    "items": {"type": "integer", "minimum": 1},
                                },
                                "outputs": {
                                    "type": "array",
                                    "items": {"type": "string"},
                                },
                                "reasoning": {"type": "string"},
                                "failure_modes": {
                                    "type": "array",
                                    "items": {"type": "string"},
                                },
                                "retries": {"type": "integer", "minimum": 0},
                            },
                        },
                    },
                    "validation_checks": {
                        "type": "array",
                        "items": {"type": "string"},
                    },
                    "expected_end_state": {"type": "string"},
                },
            },
            "negatives": {
                "type": "object",
                "properties": {
                    "confusers": {"type": "array", "items": {"type": "string"}},
                },
            },
            "metadata": {
                "type": "object",
                "additionalProperties": False,
                "required": [
                    "domain",
                    "difficulty",
                    "source",
                    "created_at",
                    "generator_model",
                    "version",
                ],
                "properties": {
                    "domain": {"type": "string"},
                    "difficulty": {
                        "type": "string",
                        "enum": ["XS", "S", "M", "L", "XL"],
                    },
                    "source": {
                        "type": "string",
                        "enum": ["synthetic", "human", "mixed"],
                    },
                    "created_at": {"type": "string"},
                    "generator_model": {"type": "string"},
                    "version": {"type": "integer"},
                },
            },
        },
    }


def list_api_names(api_catalog: Dict[str, Any]) -> List[str]:
    return [a.get("name", "") for a in api_catalog.get("apis", [])]


def validate_plan_against_catalog(
    plan: Dict[str, Any], api_catalog: Dict[str, Any]
) -> List[str]:
    """Lightweight static checks to catch obvious issues."""
    errors: List[str] = []
    api_map = {a["name"]: a for a in api_catalog.get("apis", []) if "name" in a}

    # Check steps reference known APIs and required params exist
    for step in plan.get("steps", []):
        api_name = step.get("api")
        if api_name not in api_map:
            errors.append(f"Unknown API referenced: {api_name}")
            continue
        spec = api_map[api_name]
        required = {p["name"] for p in spec.get("parameters", []) if p.get("required")}
        provided = set((step.get("arguments") or {}).keys())
        missing = required - provided
        if missing:
            errors.append(
                f"Step {step.get('index')} missing required params for {api_name}: {sorted(missing)}"
            )

    # Check indices and dependencies are consistent
    indices = [s.get("index") for s in plan.get("steps", [])]
    if sorted(indices) != list(range(1, len(indices) + 1)):
        errors.append("Step indices must start at 1 and be consecutive.")

    for s in plan.get("steps", []):
        for dep in s.get("depends_on", []) or []:
            if dep >= s.get("index", 1):
                errors.append(
                    f"Step {s.get('index')} depends_on future or same step {dep}."
                )

    return errors


def make_instruction_prompt() -> str:
    return (
        "You are a senior task-planning data engineer. Given an API catalog, "
        "you will CREATE ONE dataset record suitable for training an LLM to map "
        "natural-language tasks to a SEQUENCE of API calls (a workflow). "
        "Rules:\n"
        "- Only use APIs that exist in the catalog; never invent endpoints or params.\n"
        "- Every step must include valid arguments and reference previous outputs via variable names when needed.\n"
        "- Prefer small, composable steps (1–5) unless the task requires more.\n"
        "- Include validation checks and at least one likely failure mode.\n"
        "- Ensure required parameters are present and values match types (use realistic but fake data).\n"
        "- Avoid PII and secrets.\n"
        "- Output STRICTLY as JSON matching the provided schema. No markdown. No extra text."
    )


VARIATIONS = [
    "edge_case_constraints: include boundary numeric values and ensure validation.",
    "multi_entity: the task involves multiple items/entities processed in a loop.",
    "missing_info: include a step to fetch or infer missing required data.",
    "fallback_flow: include an alternate step if a preferred API fails or returns empty.",
    "idempotency: ensure retries and dedupe where endpoint is non-idempotent.",
    "cost_aware: minimize number of API calls while meeting the goal.",
    "compliance: add a validation check for policy/eligibility constraints.",
    "performance: parallelize independent steps and then join results (express via depends_on).",
]


def build_messages(
    api_catalog: Dict[str, Any], seed_task: Optional[str], variation: str
) -> List[Dict[str, Any]]:
    system = {"role": "system", "content": make_instruction_prompt()}
    # Include the API catalog as developer content to discourage hallucinations
    dev = {
        "role": "developer",
        "content": json.dumps({"api_catalog": api_catalog}, ensure_ascii=False),
    }
    if seed_task:
        user_text = (
            f"Create one record for the following task: {seed_task}\n"
            f"Apply variation: {variation}"
        )
    else:
        user_text = (
            "Create one record for a realistic task enabled by the catalog. "
            f"Apply variation: {variation}"
        )
    user = {"role": "user", "content": user_text}
    return [system, dev, user]


def choose_variation(i: int) -> str:
    random.seed(i * 9973)
    return random.choice(VARIATIONS)


@retry(
    wait=wait_random_exponential(min=1, max=60),
    stop=stop_after_attempt(6),
    retry=retry_if_exception_type(
        (RateLimitError, APIError, APITimeoutError, APIConnectionError)
    ),
    reraise=True,
)
def call_model(
    client: AzureOpenAI, model: str, messages: List[Dict[str, Any]], schema: Dict[str, Any]
) -> Tuple[Dict[str, Any], Dict[str, int]]:
    """Call the model via chat.completions; request JSON output per schema when supported.

    Returns a tuple: (parsed_json, usage_dict) where usage_dict contains
    prompt_tokens, completion_tokens, total_tokens when available.
    """
    # Map 'developer' -> 'system' for Azure compatibility
    chat_messages = [
        m if m.get("role") != "developer" else {"role": "system", "content": m.get("content", "")}
        for m in messages
    ]

    # Try with response_format first; some Azure deployments support it
    try:
        resp = client.chat.completions.create(
            model=model,
            messages=chat_messages,
            response_format={
                "type": "json_schema",
                "json_schema": {"name": "task_execution_record", "schema": schema},
                "strict": True,
            },
        )
        content = resp.choices[0].message.content
        usage = getattr(resp, "usage", None)
        usage_dict = {
            "prompt_tokens": getattr(usage, "prompt_tokens", 0) if usage else 0,
            "completion_tokens": getattr(usage, "completion_tokens", 0) if usage else 0,
            "total_tokens": getattr(usage, "total_tokens", 0) if usage else 0,
        }
        return json.loads(content), usage_dict
    except Exception:
        # Retry without response_format; ask the model to output raw JSON
        sys_prompt = chat_messages[0]["content"] if chat_messages and chat_messages[0]["role"] == "system" else ""
        forced = sys_prompt + "\nReturn ONLY compact JSON matching the schema, no markdown or prose."
        chat_messages[0] = {"role": "system", "content": forced}
        resp = client.chat.completions.create(
            model=model,
            messages=chat_messages,
        )
        content = resp.choices[0].message.content
        # Best-effort JSON extraction
        try:
            parsed = json.loads(content)
        except json.JSONDecodeError:
            # Try to find a JSON block
            start = content.find("{")
            end = content.rfind("}")
            if start != -1 and end != -1 and end > start:
                parsed = json.loads(content[start : end + 1])
            else:
                raise
        usage = getattr(resp, "usage", None)
        usage_dict = {
            "prompt_tokens": getattr(usage, "prompt_tokens", 0) if usage else 0,
            "completion_tokens": getattr(usage, "completion_tokens", 0) if usage else 0,
            "total_tokens": getattr(usage, "total_tokens", 0) if usage else 0,
        }
        return parsed, usage_dict


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--api-catalog", required=True, help="Path to API catalog JSON")
    parser.add_argument("--output", default="dataset.jsonl", help="Output JSONL path")
    parser.add_argument(
        "--samples", type=int, default=500, help="Number of records to generate"
    )
    parser.add_argument(
        "--model",
        default="gpt-5-mini",
        help=(
            "Azure OpenAI deployment name to use (e.g., your 'gpt-4o' deployment name)."
        ),
    )
    parser.add_argument(
        "--concurrency",
        type=int,
        default=5,
        help="Number of parallel requests to run (suggest 5-10; tune for throttling).",
    )
    parser.add_argument(
        "--seed-task", default=None, help="Optional single seed task string"
    )
    parser.add_argument(
        "--sft-output",
        default=None,
        help="Optional path to write an instruction/input/response JSONL converted from the dataset.",
    )
    parser.add_argument(
        "--sft-include-failed",
        action="store_true",
        help="Include failed/empty plan rows in SFT output (default: skip).",
    )
    parser.add_argument(
        "--sft-source",
        default=None,
        help="Optional path to read the source JSONL for SFT conversion. Defaults to --output.",
    )
    args = parser.parse_args()

    # Validate Azure OpenAI environment
    endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
    api_key = os.getenv("AZURE_OPENAI_API_KEY") or os.getenv("AZURE_OPENAI_KEY")
    api_version = os.getenv("AZURE_OPENAI_API_VERSION")
    if not endpoint or not api_key:
        raise RuntimeError(
            "Missing Azure OpenAI configuration. Set AZURE_OPENAI_ENDPOINT and AZURE_OPENAI_API_KEY (or AZURE_OPENAI_KEY)."
        )

    # Instantiate Azure OpenAI client using base_url /openai/vi/
    client = AzureOpenAI(
        azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
        api_key=os.getenv("AZURE_OPENAI_API_KEY"),
        api_version="2024-10-21",
    )

    api_catalog = load_json(args.api_catalog)
    schema = build_dataset_schema()

    records: List[Dict[str, Any]] = []
    total_prompt_tokens = 0
    total_completion_tokens = 0
    total_tokens = 0
    overall_start = time.perf_counter()
    # Worker function for a single record
    def generate_one(idx: int) -> Tuple[int, Dict[str, Any], Dict[str, int], float]:
        variation_local = choose_variation(idx)
        msgs_local = build_messages(api_catalog, args.seed_task, variation_local)
        start = time.perf_counter()
        usage_local: Dict[str, int] = {"prompt_tokens": 0, "completion_tokens": 0, "total_tokens": 0}
        try:
            item_local, usage_local = call_model(client, args.model, msgs_local, schema)
        except Exception as e:
            item_local = {
                "task_id": str(uuid.uuid4()),
                "task_nl": f"FAILED_GENERATION: {type(e).__name__}: {e}",
                "plan": {"steps": [], "expected_end_state": "error"},
                "metadata": {
                    "domain": "unknown",
                    "difficulty": "S",
                    "source": "synthetic",
                    "created_at": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
                    "generator_model": args.model,
                    "version": 1,
                },
            }
        # Validate and finalize
        try:
            validate(item_local, schema)
        except ValidationError as ve:
            item_local["plan"] = item_local.get("plan", {}) or {}
            item_local["plan"]["validation_checks"] = (
                item_local["plan"].get("validation_checks") or []
            ) + [f"SCHEMA_VALIDATION_ERROR: {ve.message[:200]}"]
        static_errors_local = validate_plan_against_catalog(item_local.get("plan", {}), api_catalog)
        if static_errors_local:
            item_local.setdefault("plan", {}).setdefault("validation_checks", [])
            item_local["plan"]["validation_checks"].extend(
                [f"STATIC_CHECK: {e}" for e in static_errors_local]
            )
        item_local.setdefault("metadata", {})
        item_local["metadata"].setdefault(
            "domain",
            ",".join(
                sorted(
                    set(
                        sum([a.get("tags", []) for a in api_catalog.get("apis", [])], [])
                    )
                )
            )
            or "unknown",
        )
        item_local["metadata"].setdefault("difficulty", random.choice(["S", "M", "L"]))
        item_local["metadata"].setdefault("source", "synthetic")
        item_local["metadata"].setdefault(
            "created_at", time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
        )
        item_local["metadata"].setdefault("generator_model", args.model)
        item_local["metadata"].setdefault("version", 1)
        item_local.setdefault("task_id", str(uuid.uuid4()))
        duration = time.perf_counter() - start
        return idx, item_local, usage_local, duration

    results: Dict[int, Dict[str, Any]] = {}
    if args.samples > 0:
        with ThreadPoolExecutor(max_workers=max(1, args.concurrency)) as ex:
            futures = {ex.submit(generate_one, i): i for i in range(args.samples)}
            with tqdm(total=args.samples, desc="Generating", leave=True) as pbar:
                for fut in as_completed(futures):
                    idx, item, row_usage, row_duration = fut.result()
                    results[idx] = item
                    total_prompt_tokens += int(row_usage.get("prompt_tokens", 0))
                    total_completion_tokens += int(row_usage.get("completion_tokens", 0))
                    total_tokens += int(row_usage.get("total_tokens", 0))
                    try:
                        tqdm.write(
                            f"Row {idx+1}: {row_duration:.3f}s, tokens: prompt={row_usage.get('prompt_tokens', 0)}, "
                            f"completion={row_usage.get('completion_tokens', 0)}, total={row_usage.get('total_tokens', 0)}"
                        )
                    except Exception:
                        print(
                            f"Row {idx+1}: {row_duration:.3f}s, tokens: prompt={row_usage.get('prompt_tokens', 0)}, "
                            f"completion={row_usage.get('completion_tokens', 0)}, total={row_usage.get('total_tokens', 0)}"
                        )
                    pbar.update(1)
        # Preserve original order in output
        records = [results[i] for i in range(args.samples)]

    if args.samples > 0:
        write_jsonl(records, args.output)
        print(f"Wrote {len(records)} records to {args.output}")

    # Final totals
    overall_duration = time.perf_counter() - overall_start
    if args.samples > 0:
        print(
            "Totals: "
            f"time={overall_duration:.3f}s, prompt_tokens={total_prompt_tokens}, "
            f"completion_tokens={total_completion_tokens}, total_tokens={total_tokens}"
        )

    # Optional: convert to instruction/input/response JSONL
    def transform_to_sft(input_jsonl: str, output_jsonl: str, include_failed: bool = False) -> Dict[str, int]:
        counts = {"written": 0, "skipped": 0}
        with open(input_jsonl, "r", encoding="utf-8") as fin, open(output_jsonl, "w", encoding="utf-8") as fout:
            for line in fin:
                line = line.strip()
                if not line:
                    continue
                try:
                    row = json.loads(line)
                except Exception:
                    counts["skipped"] += 1
                    continue
                # Prefer AGF-style mapping when present: description -> instruction, user_question -> input, workflow -> response
                has_agf_fields = ("description" in row) or ("workflow" in row)
                if has_agf_fields:
                    instruction = row.get("description") or row.get("title") or ""
                    # Per requirement: map input to the generated title, with sensible fallbacks
                    input_val = (
                        row.get("title")
                        or (row.get("input_spec") or {}).get("user_question")
                        or (row.get("input_example") or {}).get("user_question")
                        or (row.get("inputs") or {}).get("user_query")
                        or row.get("nl_task_example")
                        or row.get("user_question")
                        or ""
                    )
                    response_val = row.get("workflow") if "workflow" in row else row
                    # Skip empty workflow unless including failed
                    wf = row.get("workflow")
                    if (not include_failed) and (wf is None or (isinstance(wf, list) and len(wf) == 0)):
                        counts["skipped"] += 1
                        continue
                    sft_record = {
                        "instruction": instruction,
                        "input": input_val,
                        "response": response_val,
                    }
                else:
                    # Fallback to generator schema: task_nl -> input, plan -> response
                    task_text = row.get("task_nl") or ""
                    plan = row.get("plan") or {}
                    steps = (plan or {}).get("steps") or []
                    failed = isinstance(task_text, str) and task_text.startswith("FAILED_GENERATION:")
                    if (not include_failed) and (failed or not steps):
                        counts["skipped"] += 1
                        continue
                    sft_record = {
                        "instruction": "Generate a valid JSON execution plan for the given task.",
                        "input": task_text,
                        "response": plan,
                    }
                fout.write(json.dumps(sft_record, ensure_ascii=False, separators=(",", ":")) + "\n")
                counts["written"] += 1
        return counts

    if args.sft_output:
        # Choose source file: if provided, use --sft-source, else default to --output
        source = args.sft_source or args.output
        conv = transform_to_sft(source, args.sft_output, include_failed=args.sft_include_failed)
        print(f"SFT: wrote {conv['written']} records to {args.sft_output} (skipped {conv['skipped']})")


if __name__ == "__main__":
    main()

#     python generate_task_execution_dataset.py \
#   --api-catalog agfdocs/HUB_FUNCTIONS_COLLECTIONS.postman_collection.json \
#   --output agfapidataset.jsonl \
#   --samples 500 \
#   --model gpt-5 \
#   --seed-task "create a rag for marketing analysis, using azure ai search"
```

- Once the file are saved. Then we need to convert to SFT format.
- Here is the code for SFT conversational style formating

```
import json

# -------- CONFIG --------
TRAIN_IN = "agftrain.jsonl"
TEST_IN = "agftest.jsonl"
TRAIN_OUT = "agftrain_out.jsonl"  # new fine-tune format
TEST_OUT = "agftest_out.jsonl"
SYSTEM_PROMPT = "You are a helpful AI assistant."
# ------------------------

def convert_record(record):
    """Convert old record into OpenAI fine-tune messages format."""
    user_content = record.get("instruction", "")
    if record.get("input"):
        user_content += "\n\nInput: " + str(record["input"])

    return {
        "messages": [
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": user_content.strip()},
            {"role": "assistant", "content": str(record.get("response", ""))}
        ]
    }

def process_file(input_path, output_path):
    with open(input_path, "r", encoding="utf-8") as infile, \
         open(output_path, "w", encoding="utf-8") as outfile:
        for line in infile:
            if not line.strip():
                continue
            record = json.loads(line)
            new_record = convert_record(record)
            outfile.write(json.dumps(new_record, ensure_ascii=False) + "\n")

# Run conversion
process_file(TRAIN_IN, TRAIN_OUT)
process_file(TEST_IN, TEST_OUT)

print(f"✅ Conversion complete! Wrote {TRAIN_OUT} and {TEST_OUT}")
```

- input from distillation is agftrain.jsonl, agftest.jsonl which is in different format.
- Now we should have agftrain_out.jsonl and agftest_out.jsonl files with the SFT format.
- Now to fine tune

### Fine-tuning the Model

- Log into Azure AI Foundry
- we are only using 350 rows for training and 50+ for validation
- All files should be JSONL format
- Now go to Fine tuning on the left menu, select Fine tune model
- Now select gpt 4.1 nano
- Now upload training data set

    ![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agffinetunegptnano-1.jpg 'RagChat')

- now upload test data set or validation

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agffinetunegptnano-2.jpg 'RagChat')

- now configure training parameters

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agffinetunegptnano-3.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agffinetunegptnano-4.jpg 'RagChat')

- Click Submit once ready.
- Now it's waiting game
- Took about - 1 hour and 45 minutes
- Once completed check the results

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agffinetunegptnano-5.jpg 'RagChat')

- Now check the metrics to see the token accuracy and loss

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agffinetunegptnano-6.jpg 'RagChat')

- Check the logs and checkpoint.
- Checkpoint is the fine tune model and we can deploy it.
- Go to Model card, Select Use this Model

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agffinetunegptnano-7.jpg 'RagChat')

- now we can try the fine-tuned model
- test it with various prompts and inputs to see how well it performs
- Validate the workflows.

## Conclusion

In this guide, we covered the process of fine-tuning a model using Azure AI Foundry. We started with data preparation, including generating a task execution dataset and converting it to the required format. Then, we walked through the fine-tuning process, from uploading the training and validation datasets to configuring the training parameters and monitoring the training progress.

By following these steps, you should be able to successfully fine-tune a model for your specific use case and validate its performance through testing and evaluation.
