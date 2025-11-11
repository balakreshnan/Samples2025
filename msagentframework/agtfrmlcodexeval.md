# Azure Open AI Codex using Responses API

## Introduction

- To implement Codex model using Azure AI Foundry Open AI gpt-5-codex model.
- To evaluate the code generation capabilities of the Codex model using Responses API.

## Prerequisites

- Azure Subscription
- Azure AI Foundry Workspace
- Deploy GPT-5-Codex model in Azure AI Foundry
- Make sure choose the region where it's available.
- create a local python environment with required packages installed.
- Python 3.13+
- Install microsoft agent framework
- Install Azure-ai-projects for Azure AI Foundry SDK
- Install openai package for Azure OpenAI SDK
- i am using open ai responsesapi to call the codex model.

## Code Example

- Import required libraries

```
import asyncio
import os
import json
from datetime import datetime
from pathlib import Path
from random import randint
from typing import Annotated, Dict, List, Any
from openai import AzureOpenAI
import requests
from agent_framework import ChatAgent, ChatMessage
from agent_framework import AgentProtocol, AgentThread, HostedMCPTool, HostedFileSearchTool, HostedVectorStoreContent
from agent_framework.azure import AzureAIAgentClient
from azure.ai.projects.aio import AIProjectClient
from azure.identity.aio import AzureCliCredential, DefaultAzureCredential
from opentelemetry.trace import SpanKind
from opentelemetry.trace.span import format_trace_id
from agent_framework.observability import get_tracer, setup_observability
from pydantic import Field
from agent_framework import AgentRunUpdateEvent, WorkflowBuilder, WorkflowOutputEvent
from typing import Final

from agent_framework import (
    AgentExecutorRequest,
    AgentExecutorResponse,
    AgentRunResponse,
    AgentRunUpdateEvent,
    ChatMessage,
    Role,
    WorkflowBuilder,
    WorkflowContext,
    WorkflowOutputEvent,
    executor,
)
import os
from azure.ai.evaluation import ToolCallAccuracyEvaluator, AzureOpenAIModelConfiguration
from azure.ai.evaluation import IntentResolutionEvaluator, TaskAdherenceEvaluator, ResponseCompletenessEvaluator

from pprint import pprint

from dotenv import load_dotenv

# Load environment variables
load_dotenv()
```

- Normalize token usage function

```
def normalize_token_usage(usage) -> dict:
    """Normalize various usage objects/dicts into a standard dict.
    Returns {'prompt_tokens', 'completion_tokens', 'total_tokens'} or {} if unavailable.
    """
    try:
        if not usage:
            return {}
        # If it's already a dict, use it directly
        if isinstance(usage, dict):
            d = usage
        else:
            # Attempt attribute access (e.g., ResponseUsage pydantic model)
            d = {}
            for name in ("prompt_tokens", "completion_tokens", "total_tokens", "input_tokens", "output_tokens"):
                val = getattr(usage, name, None)
                if val is not None:
                    d[name] = val

        prompt = int(d.get("prompt_tokens", d.get("input_tokens", 0)) or 0)
        completion = int(d.get("completion_tokens", d.get("output_tokens", 0)) or 0)
        total = int(d.get("total_tokens", prompt + completion) or 0)
        return {"prompt_tokens": prompt, "completion_tokens": completion, "total_tokens": total}
    except Exception:
        return {}
```

- now the main code to process using gpt-5-codex model

```
def get_chat_response_gpt5_codex_response(query: str) -> str:
    returntxt = ""

    responseclient = AzureOpenAI(
        base_url = os.getenv("AZURE_OPENAI_ENDPOINT") + "/openai/v1/",  
        api_key= os.getenv("AZURE_OPENAI_KEY"),
        api_version="preview"
        )
    deployment = "gpt-5-codex"


    prompt = """You are a coding expert AI assistant.
    You will be given a programming task description and a list of tasks to complete.
    Your job is to generate code that accomplishes the tasks described.
    Also create the code by modular functions with comments.    
    """

    # Some new parameters!  
    response = responseclient.responses.create(
        input=prompt.format(
            query=query,
            tasks_data=json.dumps(tasks_data, indent=2)
        ),
        model=deployment,
        reasoning={
            "effort": "medium",
            "summary": "auto" # auto, concise, or detailed 
        },
        text={
            "verbosity": "medium" # New with GPT-5 models
        }
    )

    # # Token usage details
    usage = normalize_token_usage(response.usage)

    # print("--------------------------------")
    # print("Output:")
    # print(output_text)
    returntxt = response.output_text

    return returntxt, usage
```

- Here is the function to run the codex evaluation

```
def evaluate_agent(query, response):
    returntxt = ""

    # query = "How is the weather in Seattle ?"
    tool_call = {
        "type": "tool_call",
        "tool_call_id": "call_CUdbkBfvVBla2YP3p24uhElJ",
        "name": "fetch_weather",
        "arguments": {"location": "Seattle"},
    }

    tool_definition = {
        "id": "fetch_weather",
        "name": "fetch_weather",
        "description": "Fetches the weather information for the specified location.",
        "parameters": {
            "type": "object",
            "properties": {"location": {"type": "string", "description": "The location to fetch weather for."}},
        },
    }
    model_config = AzureOpenAIModelConfiguration(
        azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
        api_key=os.environ["AZURE_OPENAI_KEY"],
        api_version=os.environ["AZURE_OPENAI_API_VERSION"],
        azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT"],
    )


    tool_call_accuracy = ToolCallAccuracyEvaluator(model_config=model_config)
    tool_response = tool_call_accuracy(query=query, tool_calls=tool_call, tool_definitions=tool_definition)
    pprint(tool_response)
    returntxt += f"Tool Call Accuracy Evaluation:\n{tool_response}\n\n"
    response_completeness_evaluator = ResponseCompletenessEvaluator(model_config=model_config)
    result = response_completeness_evaluator(
        response=response,
        ground_truth=groundtruth,
    )
    pprint(result)
    returntxt += f"Response Completeness Evaluation:\n{result}\n\n"
    intent_resolution_evaluator = IntentResolutionEvaluator(model_config)
    # Success example. Intent is identified and understood and the response correctly resolves user intent
    result = intent_resolution_evaluator(
        query=query,
        response=response,
    )
    pprint(result)
    returntxt += f"Intent Resolution Evaluation:\n{result}\n\n"
    task_adherence_evaluator = TaskAdherenceEvaluator(model_config)
    # TaskAdherenceEvaluator requires a conversation format
    conversation = {
        "messages": [
            {"role": "user", "content": query},
            {"role": "assistant", "content": response}
        ]
    }
    result = task_adherence_evaluator(
        conversation=conversation,
    )
    pprint(result)
    returntxt += f"Task Adherence Evaluation:\n{result}\n\n"

    return returntxt
```

- now the main function to call above methods

```
def main():
    query = """Create a game for humanoids learning python code.
            """

    response, usage = get_chat_response_gpt5_codex_response(query)

    print("Response:")
    print(response)
    print("Token Usage:")
    print(usage)
    print("===============================\n")
    evaluation = evaluate_agent(query, response)
    print("Evaluation Results:")
    print(evaluation)

if __name__ == "__main__":
    main()
```

## Conclusion

- In this example, we demonstrated how to use the Azure OpenAI Codex model (gpt-5-codex) via the Responses API to generate code based on a programming task description.
- We also evaluated the generated code using various evaluation metrics such as Tool Call Accuracy, Response Completeness, Intent Resolution, and Task Adherence.
- This approach can be extended to various coding tasks and can help in automating code generation and evaluation processes.
- Only for development and testing purposes.