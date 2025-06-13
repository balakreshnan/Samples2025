# New Azure AI Foundry Resource - Evaluation. RedTeam, Agent Evaluation

## Azure AI Foundry - Evaluation, Red Team, Agent Evaluation as part of GenAIOps and Development

## Introduction

- Azure AI Foundry new resource allows us to create Gen AI application and Agent AI Agents.
- Which is python based SDK to create and manage AI applications.
- Now we can implement Gen AI Ops to deploy and manage AI applications.
- What lot's of folks forget is the responsible ai part and cybersecurity with respect to Language Models.
- This is where the new Evaluation and RedTeam resource comes into play.
- This resource allows us to evaluate and test our AI applications for security vulnerabilities, biases, and other potential issues.
- It provides a framework for conducting red teaming exercises and evaluating the performance of AI models in a controlled environment.
- This is crucial for ensuring that AI applications are secure, reliable, and compliant with ethical standards.
- We can add this to Gen AI Ops as steps to validate our AI applications before deploying them to production.
- Also show case how to use Trace to log application lineage and telemetry data for monitoring and debugging purposes.

## Prerequisites

- Azure subscription
- Azure AI Foundry resource
- Deploy gpt 4.1, gpt 4o models
- Visual studio code
- Python 3.10, 3.11, 3.12 works best
- create a virtual environment
- Install the packages from requirements.txt
- here is the requirements.txt file for the new Azure AI Foundry resource:

```
azure-ai-projects
azure-ai-agents
azure-ai-evaluation
azure-ai-evaluation[redteam]
azure-ai-inference
streamlit
openai
uv
python-dotenv
opentelemetry-sdk
opentelemetry-api
azure-identity
azure-search-documents
azure-ai-inference[prompts]
azure-monitor-opentelemetry
speech-recognition
pydub
```

## Getting Started

### Steps

- Let us create a new python file called agenticai.py
- Let's import the necessary libraries and modules:

```python
import asyncio
from datetime import datetime
import time
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import (
    RedTeam,
    AzureOpenAIModelConfiguration,
    AttackStrategy,
    RiskCategory,
)
import os, json
import pandas as pd
from typing import Any, Callable, Set, Dict, List, Optional
from azure.ai.agents.models import CodeInterpreterTool, FunctionTool, ToolSet
from azure.ai.projects.models import (
    EvaluatorConfiguration,
    EvaluatorIds,
)
from azure.ai.projects.models import (
    Evaluation,
    InputDataset
)
from azure.ai.evaluation import AIAgentConverter, IntentResolutionEvaluator
from azure.ai.evaluation import (
    ToolCallAccuracyEvaluator,
    AzureOpenAIModelConfiguration,
    IntentResolutionEvaluator,
    TaskAdherenceEvaluator,
    ResponseCompletenessEvaluator,
    ContentSafetyEvaluator,
    RelevanceEvaluator,
    CoherenceEvaluator,
    GroundednessEvaluator,
    FluencyEvaluator,
    SimilarityEvaluator,
    ViolenceEvaluator,
    SexualEvaluator,
    SelfHarmEvaluator,
    HateUnfairnessEvaluator,
    RetrievalEvaluator,
    BleuScoreEvaluator, GleuScoreEvaluator, RougeScoreEvaluator, MeteorScoreEvaluator, RougeType,
    ProtectedMaterialEvaluator, IndirectAttackEvaluator, RetrievalEvaluator, GroundednessProEvaluator,
    F1ScoreEvaluator
)
from pprint import pprint
# specific to agentic workflows
from azure.ai.evaluation import IntentResolutionEvaluator, TaskAdherenceEvaluator, ToolCallAccuracyEvaluator 
# other quality as well as risk and safety metrics
from azure.ai.evaluation import RelevanceEvaluator, CoherenceEvaluator, CodeVulnerabilityEvaluator, ContentSafetyEvaluator, IndirectAttackEvaluator, FluencyEvaluator
from azure.ai.projects.models import ConnectionType
from pathlib import Path
from opentelemetry import trace
# from config import get_logger
from azure.ai.evaluation.red_team import RedTeam, RiskCategory
from openai import AzureOpenAI
from azure.ai.evaluation import evaluate
from azure.ai.evaluation import GroundednessEvaluator, AzureOpenAIModelConfiguration
from azure.ai.agents.models import ConnectedAgentTool, MessageRole
from azure.ai.agents.models import AzureAISearchTool, AzureAISearchQueryType, MessageRole, ListSortOrder, ToolDefinition, FilePurpose, FileSearchTool
from utils import send_email
from user_logic_apps import AzureLogicAppTool, create_send_email_function
from azure.ai.agents.models import ListSortOrder

from dotenv import load_dotenv

load_dotenv()
import logging
```

- I am loading all the environment variables from the .env file using the `load_dotenv()` function.
- Below is a sample .env file that you can use to store your Azure AI Foundry resource details and other configurations:
- Make sure to replace the placeholders with your actual values.

```
PROJECT_ENDPOINT="https://xxxxxxxxx.services.ai.azure.com/api/projects/projectname"
MODEL_ENDPOINT="https://xxxxxxxxx.services.ai.azure.com"
MODEL_API_KEY="xxxxxxxxxxxxxxxxxxxxxxxx"
MODEL_DEPLOYMENT_NAME="gpt-4.1"
AZURE_AI_PROJECT="https://xxxxxxxxxxxxxxx.services.ai.azure.com/api/projects/projectname"
AZURE_SUBSCRIPTION_ID="xxxxxxxxxx"
AZURE_RESOURCE_GROUP="resourcegroupname"
AZURE_PROJECT_NAME="projectname"
AZURE_OPENAI_KEY="xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
AZURE_OPENAI_ENDPOINT="https://xxxxxxxx.cognitiveservices.azure.com/"
AZURE_OPENAI_DEPLOYMENT="gpt-4.1"
AZURE_OPENAI_DEPLOYMENT_MODELROUTER="model-router"
AZURE_API_VERSION="2024-10-01-preview"
AZURE_OPENAI_ENDPOINT_REDTEAM="https://xxxxxxxx.cognitiveservices.azure.com/openai/deployments/gpt-4.1/chat/completions?api-version=2025-01-01-preview"
SEARCH_ENDPOINT="xxxxxx"
```

- COnfigure project settings and logging

```
endpoint = os.environ["PROJECT_ENDPOINT"] # Sample : https://<account_name>.services.ai.azure.com/api/projects/<project_name>
model_endpoint = os.environ["MODEL_ENDPOINT"] # Sample : https://<account_name>.services.ai.azure.com
model_api_key= os.environ["MODEL_API_KEY"]
model_deployment_name = os.environ["MODEL_DEPLOYMENT_NAME"] # Sample : gpt-4o-mini
os.environ["AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED"] = "true" 

# Create the project client (Foundry project and credentials)
project_client = AIProjectClient(
    endpoint=endpoint,
    credential=DefaultAzureCredential(),
)

client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"), 
  api_key=os.getenv("AZURE_OPENAI_KEY"),  
  api_version="2024-10-21",
)

from azure.monitor.opentelemetry import configure_azure_monitor
connection_string = project_client.telemetry.get_connection_string()

if not connection_string:
    print("Application Insights is not enabled. Enable by going to Tracing in your Azure AI Foundry project.")
    exit()

configure_azure_monitor(connection_string=connection_string) #enable telemetry collection

from opentelemetry import trace
tracer = trace.get_tracer(__name__)
```

- Above is also enabling tracing and telemetry collection for the Azure AI Foundry project.
- Now lets write the evalutaion functions to evaluate the AI applications and agents.
- I am evaluating groundedness, relevance, coherence, fluency, content safety, similarity, retrieval, and other metrics.
- I am also evaluating safety and risk metrics like hate and unfairness, protected material, indirect attacks, and groundedness pro.
- Here is the code to evaluate the AI applications and agents using the Azure AI Foundry Evaluation.
- There is also data set is needed, i created a sample dataset called `datarfp.jsonl` which you can use to test the evaluation.
- datarfp.jsonl is a JSON Lines file that contains the data for evaluation.

```
{
    query: "What is the capital of France?",
    response: "The capital of France is Paris.",
    context: "France is a country in Europe. Paris is its capital city.",
    ground_truth: "The capital of France is Paris."
}
```

- Please create your own dataset based on your requirements.
- Now the actual code to evaluate the AI applications and agents using the Azure AI Foundry Evaluation:

```
def eval()-> str:
    returntxt = ""
    model_config = AzureOpenAIModelConfiguration(
        azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT"),
        api_key=os.environ.get("AZURE_OPENAI_KEY"),
        azure_deployment=os.environ.get("AZURE_OPENAI_DEPLOYMENT"),
        api_version=os.environ.get("AZURE_API_VERSION"),
    )

    credential = DefaultAzureCredential()

    azure_ai_project = os.environ.get("PROJECT_ENDPOINT")
    azure_ai_project_dict = os.environ.get("PROJECT_ENDPOINT")

    # Initializing Groundedness and Groundedness Pro evaluators
    # groundedness_eval = GroundednessEvaluator(model_config)
    # instantiate an evaluator with image and multi-modal support
    content_safety_evaluator = ContentSafetyEvaluator(credential=DefaultAzureCredential(), azure_ai_project=azure_ai_project)
    relevance_evaluator = RelevanceEvaluator(model_config)
    coherence_evaluator = CoherenceEvaluator(model_config)
    groundedness_evaluator = GroundednessEvaluator(model_config)
    fluency_evaluator = FluencyEvaluator(model_config)
    similarity_evaluator = SimilarityEvaluator(model_config)
    retrieval_evaluator = RetrievalEvaluator(model_config)
    groundnesspro_evaluator = GroundednessProEvaluator(azure_ai_project=azure_ai_project_dict, credential=credential)

    f1_evaluator = F1ScoreEvaluator()
    bleu_evaluator = BleuScoreEvaluator()
    gleu_evaluator = GleuScoreEvaluator()
    meteor_evaluator = MeteorScoreEvaluator(alpha=0.8)
    rouge_evaluator = RougeScoreEvaluator(rouge_type=RougeType.ROUGE_4)
    indirect_attack_eval = IndirectAttackEvaluator(azure_ai_project=azure_ai_project_dict, credential=credential)
    protected_material_eval = ProtectedMaterialEvaluator(azure_ai_project=azure_ai_project_dict, credential=credential)
    hate_unfairness_eval = HateUnfairnessEvaluator(azure_ai_project=azure_ai_project_dict, credential=credential)

    result = evaluate(
        data="datarfp.jsonl", # provide your data here
        evaluators={
            "content_safety": content_safety_evaluator,
            "coherence": coherence_evaluator,
            "relevance": relevance_evaluator,
            "groundedness": groundedness_evaluator,
            "fluency": fluency_evaluator,
        #    "similarity": similarity_evaluator,
            "f1": f1_evaluator,
            "bleu": bleu_evaluator,
            "gleu": gleu_evaluator,
            "meteor": meteor_evaluator,
            "rouge": rouge_evaluator,
            "indirect_attack": indirect_attack_eval,
            "protected_material": protected_material_eval,
            "hate_unfairness": hate_unfairness_eval,
            "retrieval": retrieval_evaluator,
            "groundnesspro": groundnesspro_evaluator,
            "similarity": similarity_evaluator,
        },        
        evaluator_config={
            "content_safety": {"query": "${data.query}", "response": "${data.response}"},
            "coherence": {"response": "${data.response}", "query": "${data.query}"},
            "relevance": {"response": "${data.response}", "context": "${data.context}", "query": "${data.query}"},
            "groundedness": {
                "response": "${data.response}",
                "context": "${data.context}",
                "query": "${data.query}",
            },
            "fluency": {"response": "${data.response}", "context": "${data.context}", "query": "${data.query}"},
            "f1": {"response": "${data.response}", "ground_truth": "${data.ground_truth}"},
            "bleu": {"response": "${data.response}", "ground_truth": "${data.ground_truth}"},
            "gleu": {"response": "${data.response}", "ground_truth": "${data.ground_truth}"},
            "meteor": {"response": "${data.response}", "ground_truth": "${data.ground_truth}"},
            "rouge": {"response": "${data.response}", "ground_truth": "${data.ground_truth}"},
            "indirect_attack": {"query": "${data.query}", "response": "${data.response}"},
            "protected_material": {"query": "${data.query}", "response": "${data.response}"},
            "hate_unfairness": {"query": "${data.query}", "response": "${data.response}"},
            "retrieval": {"query": "${data.query}", "context": "${data.context}"},
            "groundnesspro": {"query": "${data.query}", "context" : "${data.context}", "response": "${data.response}"},
            "similarity": {"query": "${data.query}", "response": "${data.response}", "ground_truth": "${data.ground_truth}"},
        },
        # Optionally provide your Azure AI Foundry project information to track your evaluation results in your project portal
        azure_ai_project = os.environ["PROJECT_ENDPOINT"],
        # Optionally provide an output path to dump a json of metric summary, row level data and metric and Azure AI project URL
        output_path="./myevalresults.json"
    )
    #returntxt = f"Completed Evaluation: {result.studio_url}"    
    returntxt = f"Completed Evaluation\n"
    return returntxt
```

- Now we can execute the code but we are going to continue with red teaming code.
- Red Team has bacic, intermediate, and advanced levels of evaluation.
- We also have to creaet callback functions to validate.
- Here is callback function

```
# A simple example application callback function that always returns a fixed response
def simple_callback(query: str) -> str:
    return "I'm an AI assistant that follows ethical guidelines. I cannot provide harmful content."
```

- Advanced callback function

```
async def advanced_callback(messages: Dict, stream: bool = False, session_state: Any = None, context: Optional[Dict] =None) -> dict:
    """A more complex callback that processes conversation history"""
    # Extract the latest message from the conversation history
    messages_list = [{"role": chat_message.role,"content": chat_message.content} for chat_message in messages]
    latest_message = messages_list[-1]["content"]
    
    # In a real application, you might process the entire conversation history
    # Here, we're just simulating different responses based on the latest message
    response = "I'm an AI assistant that follows safety guidelines. I cannot provide harmful content."
    
    # Format the response to follow the openAI chat protocol format
    formatted_response = {
        "content": response,
        "role": "assistant"
    }
    
    return {"messages": [formatted_response]}
```

- Now the actual code for redteaming using the Azure AI Foundry Evaluation and RedTeam:

```
async def redteam() -> str:
    returntxt = ""
                
    ## Using Azure AI Foundry Hub project
    azure_ai_project = {
        "subscription_id": os.environ.get("AZURE_SUBSCRIPTION_ID"),
        "resource_group_name": os.environ.get("AZURE_RESOURCE_GROUP"),
        "project_name": os.environ.get("AZURE_PROJECT_NAME"),
    }
    ## Using Azure AI Foundry project, example: AZURE_AI_PROJECT=https://your-account.services.ai.azure.com/api/projects/your-project
    azure_ai_project = os.environ.get("AZURE_AI_PROJECT")


    # Specifying risk categories and number of attack objectives per risk categories you want the AI Red Teaming Agent to cover
    red_team_agent = RedTeam(
        azure_ai_project=azure_ai_project, # required
        credential=DefaultAzureCredential(), # required
        risk_categories=[ # optional, defaults to all four risk categories
            RiskCategory.Violence,
            RiskCategory.HateUnfairness,
            RiskCategory.Sexual,
            RiskCategory.SelfHarm
        ], 
        num_objectives=5, # optional, defaults to 10
    )
    # Runs a red teaming scan on the simple callback target
    red_team_result = await red_team_agent.scan(target=simple_callback)

    # Define a model configuration to test
    azure_oai_model_config = {
        "azure_endpoint": os.environ.get("AZURE_OPENAI_ENDPOINT_REDTEAM"),
        "azure_deployment": os.environ.get("AZURE_OPENAI_DEPLOYMENT"),
        "api_key": os.environ.get("AZURE_OPENAI_KEY"),
    }
    # # Run the red team scan called "Intermediary-Model-Target-Scan"
    result = await red_team_agent.scan(
        target=azure_oai_model_config, scan_name="Intermediary-Model-Target-Scan", attack_strategies=[AttackStrategy.Flip]
    )
    returntxt += str(result)

    # Create the RedTeam instance with all of the risk categories with 5 attack objectives generated for each category
    model_red_team = RedTeam(
        azure_ai_project=azure_ai_project,
        credential=DefaultAzureCredential(),
        risk_categories=[RiskCategory.Violence, RiskCategory.HateUnfairness, RiskCategory.Sexual, RiskCategory.SelfHarm],
        num_objectives=2,
    )

    # Run the red team scan with multiple attack strategies
    advanced_result = await model_red_team.scan(
        target=advanced_callback,
        scan_name="Advanced-Callback-Scan",
        attack_strategies=[
            AttackStrategy.EASY,  # Group of easy complexity attacks
            AttackStrategy.MODERATE,  # Group of moderate complexity attacks
            # AttackStrategy.CHARACTER_SPACE,  # Add character spaces
            #AttackStrategy.ROT13,  # Use ROT13 encoding
            #AttackStrategy.UnicodeConfusable,  # Use confusable Unicode characters
            #AttackStrategy.CharSwap,  # Swap characters in prompts
            #AttackStrategy.Morse,  # Encode prompts in Morse code
            #AttackStrategy.Leetspeak,  # Use Leetspeak
            #AttackStrategy.Url,  # Use URLs in prompts
            #AttackStrategy.Binary,  # Encode prompts in binary
            # AttackStrategy.Compose([AttackStrategy.BASE64, AttackStrategy.ROT13]),  # Use two strategies in one 
        ],
        output_path="./Advanced-Callback-Scan.json",
    )

    returntxt += str(advanced_result)

    #returntxt += f"Red Team scan completed with status: {red_team_agent.ai_studio_url}\n"
        
    return returntxt
```

- Now we are also going to add agents evaluation using the Azure AI Foundry Evaluation.
- Agentic eval needs tools calling in the data format.
- Please check the current documentation for the latest updates on the tools and their usage.

```
def agent_eval() -> str:
    returntxt = ""
    user_functions: Set[Callable[..., Any]] = {
        fetch_weather,
    }

    # Adding Tools to be used by Agent 
    functions = FunctionTool(user_functions)

    toolset = ToolSet()
    toolset.add(functions)


    project_client = AIProjectClient(
        endpoint=endpoint,
        credential=DefaultAzureCredential(),
    )

    AGENT_NAME = "Agentic Eval Assistant"

    # Add Tools to be used by Agent
    functions = FunctionTool(user_functions)

    toolset = ToolSet()
    toolset.add(functions)

    # To enable tool calls executed automatically
    project_client.agents.enable_auto_function_calls(tools=toolset)
    agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name=AGENT_NAME,
        instructions="You are a helpful assistant",
        toolset=toolset,
    )

    print(f"Created agent, ID: {agent.id}")
    # https://github.com/Azure-Samples/azureai-samples/blob/main/scenarios/evaluate/Supported_Evaluation_Metrics/Agent_Evaluation/Evaluate_Azure_AI_Agent_Quality.ipynb
    # Create a thread for communication
    thread = project_client.agents.threads.create()
    print(f"Created thread, ID: {thread.id}")
    # Create message to thread

    MESSAGE = "Can you email me weather info for Seattle ?"

    # Add a message to the thread
    message = project_client.agents.messages.create(
        thread_id=thread.id,
        role="user",  # Role of the message sender
        content=MESSAGE,  # Message content
    )
    print(f"Created message, ID: {message['id']}")
    # Create and process an agent run
    run = project_client.agents.runs.create_and_process(thread_id=thread.id, agent_id=agent.id)
    print(f"Run finished with status: {run.status}")
    
    # Check if the run failed
    if run.status == "failed":
        print(f"Run failed: {run.last_error}")
    
    # Fetch and log all messages
    messages = project_client.agents.messages.list(thread_id=thread.id)
    for message in messages:
        print(f"Role: {message.role}, Content: {message.content}")
    # Initialize the converter that will be backed by the project.
    converter = AIAgentConverter(project_client)

    thread_id = thread.id
    run_id = run.id
    file_name = "evaluation_input_data.jsonl"

    # Get a single agent run data
    evaluation_data_single_run = converter.convert(thread_id=thread_id, run_id=run_id)

    # Run this to save thread data to a JSONL file for evaluation
    # Save the agent thread data to a JSONL file
    # evaluation_data = converter.prepare_evaluation_data(thread_ids=thread_id, filename=<>)
    # print(json.dumps(evaluation_data, indent=4))
    model_config = AzureOpenAIModelConfiguration(
        azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
        api_key=os.environ["AZURE_OPENAI_KEY"],
        api_version=os.environ["AZURE_API_VERSION"],
        azure_deployment=os.environ["MODEL_DEPLOYMENT_NAME"],
    )

    azure_ai_project = os.environ["PROJECT_ENDPOINT"]

    intent_resolution = IntentResolutionEvaluator(model_config=model_config)
    tool_call_accuracy = ToolCallAccuracyEvaluator(model_config=model_config)
    task_adherence = TaskAdherenceEvaluator(model_config=model_config)
    #response_completeness_evaluator = CompletenessEvaluator(model_config=model_config, azure_ai_project=azure_ai_project)
    
    response = evaluate(
        data=file_name,
        evaluators={
            "tool_call_accuracy": tool_call_accuracy,
            "intent_resolution": intent_resolution,
            "task_adherence": task_adherence,
            #"response_completeness": response_completeness_evaluator,
        },
        azure_ai_project=os.environ["PROJECT_ENDPOINT"],
    )
    pprint(f'AI Foundary URL: {response.get("studio_url")}')
    # average scores across all runs
    pprint(response["metrics"])
    returntxt = str(response["metrics"])

    # Delete the agent when done
    project_client.agents.delete_agent(agent.id)
    print("Deleted agent")
    
    return returntxt
```

- Now lets create the main function to run the evaluation and red teaming code.

```
async def main():
    print("Starting Evaluation and Red Teaming...")
    eval_result = eval()
    print(eval_result)
    
    redteamrs = asyncio.run(redteam())
    print(redteamrs)

    agent_eval_result = agent_eval()
    print(agent_eval_result)
```

### Evaluation of Gen AI Applciation

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundrynew2025-1.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundrynew2025-2.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundrynew2025-3.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundrynew2025-4.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundrynew2025-5.jpg 'RagChat')

### Red Teaming of Gen AI Applciation

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundrynew2025-6.jpg 'RagChat')

#### Basic Red Teaming

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundrynew2025-7.jpg 'RagChat')

#### Intermediate Red Teaming

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundrynew2025-8.jpg 'RagChat')

#### Advanced Red Teaming

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundrynew2025-9.jpg 'RagChat')

### Agent Evaluation

- Now we are going to see the output of the agent evaluation.

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundrynew2025-10.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundrynew2025-11.jpg 'RagChat')

## Conclusion

- The above technologies should be included in the development phase to evaluate the AI applications and agents.
- Then it can also be used in deployment phase to validate the AI applications and agents.
- There are always the debate on being realtime and real world scenarios.
- But realtime also incoporate latency and performance issues.
- There are few like content safety and hate and unfairness which can be evaluated in realtime.
- Red Team agents even allows red team team member to create their own custom attacks and strategies.
- This tutorial is only for learning and skilling purpose.