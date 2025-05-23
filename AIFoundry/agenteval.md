# Evaluation for Agents using Azure AI Foundry

## Introduction

- Create a simple Rag Agent using Azure AI Foundry.
- Agent is using own data set which is open source data set.
- Then now we will evaluate the agent using Azure AI Foundry.

## Prerequisites

- Azure Subscription
- Azure AI Foundry
- Azure Open AI
- Visual Studio Code
- Python 3.11 Environment
- Set the environment variable

## Steps

- Import the required libraries
- please install the required libraries

```
import base64
from datetime import time
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
import json
import os
import io
import shutil
import smtplib
import PyPDF2
from PIL import Image
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import FileSearchTool, MessageAttachment, FilePurpose
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from azure.ai.projects.models import AzureAISearchTool, AzureAISearchQueryType
import os
from docx import Document
import docx
import fitz
from openai import AzureOpenAI
from pptx import Presentation
from pptx.util import Inches
from typing import Any, Callable, Set, Dict, List, Optional
from azure.ai.evaluation import evaluate, AzureAIProject, AzureOpenAIModelConfiguration, F1ScoreEvaluator
from azure.ai.evaluation import RelevanceEvaluator
from azure.ai.evaluation import (
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
)

from azure.ai.evaluation import BleuScoreEvaluator, GleuScoreEvaluator, RougeScoreEvaluator, MeteorScoreEvaluator, RougeType
from azure.ai.projects.models import FunctionTool, RequiredFunctionToolCall, SubmitToolOutputsAction, ToolOutput, ToolSet
from azure.ai.evaluation import ProtectedMaterialEvaluator, IndirectAttackEvaluator, RetrievalEvaluator, GroundednessProEvaluator
from typing import Any, Callable, Set, Dict, List, Optional
from azure.ai.evaluation.red_team import RedTeam, RiskCategory, AttackStrategy
from azure.ai.evaluation import (
    ToolCallAccuracyEvaluator,
    AzureOpenAIModelConfiguration,
    IntentResolutionEvaluator,
    TaskAdherenceEvaluator,
)
from pprint import pprint
import time  # Import the time module
import streamlit as st
from azure.ai.evaluation import AIAgentConverter

 

from dotenv import load_dotenv
load_dotenv()

project_client = AIProjectClient.from_connection_string(
    credential=DefaultAzureCredential(),
    conn_str=os.getenv("PROJECT_CONNECTION_STRING"),
)

client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"), 
  api_key=os.getenv("AZURE_OPENAI_API_KEY"),  
  api_version="2024-10-21",
)

model_name = os.getenv("AZURE_OPENAI_DEPLOYMENT")

# Initialize the converter
converter = AIAgentConverter(project_client)
```

- now set up the agent for Rag search with your own data set
- Make sure the index is available in the Azure AI Foundry
- i used the Azure AI foundry UI to create the index
- the index name is constructionrfpdocs1
- These are my RFP documents

```
def aisearch_process_agent(query: str) -> str:

    # Extract the connection list.
    conn_list = project_client.connections._list_connections()["value"]
    conn_id = ""

    # Search in the metadata field of each connection in the list for the azure_ai_search type and get the id value to establish the variable
    for conn in conn_list:
        metadata = conn["properties"].get("metadata", {})
        if metadata.get("type", "").upper() == "AZURE_AI_SEARCH":
            conn_id = conn["id"]
            break

    # Initialize agent AI search tool and add the search index connection ID and index name
    # TO DO: replace <your-index-name> with the name of the index you want to use
    ai_search = AzureAISearchTool(index_connection_id=conn_id, index_name="constructionrfpdocs1",
    query_type=AzureAISearchQueryType.SIMPLE)

    agent = project_client.agents.create_agent(
        model="gpt-4o",
        name="ai_search_agent",
        instructions="You are a helpful assistant",
        tools=ai_search.definitions,
        tool_resources = ai_search.resources,
    )
    print(f"Created agent, ID: {agent.id}")

    # Create a thread
    thread = project_client.agents.create_thread()
    print(f"Created thread, thread ID: {thread.id}")
    
    # Create a message
    message = project_client.agents.create_message(
        thread_id=thread.id,
        role="user",
        content=query,
    )
    print(f"Created message, message ID: {message.id}")
        
    # Run the agent
    run = project_client.agents.create_and_process_run(thread_id=thread.id, agent_id=agent.id)
    print(f"Run finished with status: {run.status}")
    
    if run.status == "failed":
        # Check if you got "Rate limit is exceeded.", then you want to get more quota
        print(f"Run failed: {run.last_error}")

    # Get messages from the thread 
    messages = project_client.agents.list_messages(thread_id=thread.id)
    #print(f"Messages: {messages}")
        
    assistant_message = ""
    for message in messages.data:
        if message["role"] == "assistant":
            assistant_message = message["content"][0]["text"]["value"]

    # Get the last message from the sender
    print(f"Assistant response: {assistant_message}")

    # Specify a file path to save agent output (which is evaluation input data)
    # filename = os.path.join(os.getcwd(), "evaluation_input_data.jsonl")
    filename = "evaluation_input_data.jsonl"

    evaluation_data = converter.prepare_evaluation_data(thread_ids=thread.id, filename=filename) 

    print(f"Evaluation data saved to {filename}")
    # print(f"Messages: {messages}")
    # rs = parse_output(messages)
    # print("Messages: ", assistant_message)
    returnstring = assistant_message
    return returnstring
```

- Run the above agent once to get the evaluation output data
- Save the output data in a file
- The file name is evaluation_input_data.jsonl
- Now we write the evaluation code

```
def agenteval(query: str) -> str:
    returntxt = ""
    #print('Citation Text:', citationtxt)
    model_config = AzureOpenAIModelConfiguration(
        azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
        api_key=os.environ["AZURE_OPENAI_API_KEY"],
        api_version=os.environ["AZURE_OPENAI_API_VERSION"],
        azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT"],
    )
    # Needed to use content safety evaluators
    azure_ai_project = {
        "subscription_id": os.environ["AZURE_SUBSCRIPTION_ID"],
        "project_name": os.environ["AZUREAI_PROJECT_NAME"],
        "resource_group_name": os.environ["AZURE_RESOURCE_GROUP"],
    }

    intent_resolution = IntentResolutionEvaluator(model_config=model_config)

    tool_call_accuracy = ToolCallAccuracyEvaluator(model_config=model_config)

    task_adherence = TaskAdherenceEvaluator(model_config=model_config)

    from azure.ai.evaluation import evaluate

    response = evaluate(
        # data="datarfpagent.jsonl",
        data="evaluation_input_data.jsonl",
        target=aisearch_process_agent,
        evaluation_name="AgentConstructionRFPEvaluation",
        evaluators={
            "tool_call_accuracy": tool_call_accuracy,
            "intent_resolution": intent_resolution,
            "task_adherence": task_adherence,
        },
        azure_ai_project=azure_ai_project,
    )
    pprint(f'AI Foundary URL: {response.get("studio_url")}')
    # average scores across all runs
    pprint(response["metrics"])

    # returntxt = response["metrics"].json()
    returntxt = json.dumps(response["metrics"])

    return returntxt
```

- Now we will run the agent and get the evaluation output
- The output will be in JSON format
- Code for running the agent

```
import json
import os
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import FileSearchTool, MessageAttachment, FilePurpose
from azure.identity import DefaultAzureCredential
from azure.ai.projects.models import AzureAISearchTool, AzureAISearchQueryType
import os
import pandas as pd
import asyncio
from utils import aisearch_process_agent

from openai import AzureOpenAI
from utils import parse_output, file_process_agent, aisearch_process_agent, getrfptopictorespond, create_word_doc, download_word_file, process_image, pdf_to_images, compare_rfq_drawings, agenteval
# from utils import redteam_agent, evalmetrics, rai_agent
from utils import evalmetrics, rai_agent, redteam_agent
# [START enable_tracing]
from opentelemetry import trace
from azure.monitor.opentelemetry import configure_azure_monitor
# import streamlit as st
from streamlit_quill import st_quill
import fitz  # PyMuPDF  
from azure.ai.inference.tracing import AIInferenceInstrumentor 

from dotenv import load_dotenv
load_dotenv()

def get_azure():

    #rs = aisearch_process_agent("Show me details on Construction management services experience we have done before?")
    #print(rs)
    agentevalresult = agenteval("Show me details on Construction management services experience we have done before?")
    print(agentevalresult)

if __name__ == "__main__":
    get_azure()
```

- Run the above code

```
python main.py
```

- Check the output of evaluation

```
{'intent_resolution.binary_aggregate': 1.0,
 'intent_resolution.intent_resolution': 5.0,
 'intent_resolution.intent_resolution_threshold': 3.0,
 'task_adherence.binary_aggregate': 1.0,
 'task_adherence.task_adherence': 4.0,
 'task_adherence.task_adherence_threshold': 3.0,
 'tool_call_accuracy.binary_aggregate': 0.0,
 'tool_call_accuracy.tool_call_accuracy_threshold': 0.8}
{"tool_call_accuracy.tool_call_accuracy_threshold": 0.8, "intent_resolution.intent_resolution": 5.0, "intent_resolution.intent_resolution_threshold": 3.0, "task_adherence.task_adherence": 4.0, "task_adherence.task_adherence_threshold": 3.0, "tool_call_accuracy.binary_aggregate": 0.0, "intent_resolution.binary_aggregate": 1.0, "task_adherence.binary_aggregate": 1.0}
```

```
======= Combined Run Summary (Per Evaluator) =======

{
    "tool_call_accuracy": {
        "status": "Completed",
        "duration": "0:00:01.356820",
        "completed_lines": 1,
        "failed_lines": 0,
        "log_path": "C:\\Users\\username\\.promptflow\\.runs\\azure_ai_evaluation_evaluators_tool_call_accuracy_20250523_091217_732590"
    },
    "intent_resolution": {
        "status": "Completed",
        "duration": "0:00:05.597802",
        "completed_lines": 1,
        "failed_lines": 0,
        "log_path": "C:\\Users\\username\\.promptflow\\.runs\\azure_ai_evaluation_evaluators_intent_resolution_20250523_091217_732590"
    },
    "task_adherence": {
        "status": "Completed",
        "duration": "0:00:07.518122",
        "completed_lines": 1,
        "failed_lines": 0,
        "log_path": "C:\\Users\\username\\.promptflow\\.runs\\azure_ai_evaluation_evaluators_task_adherence_20250523_091217_733587"
    }
}

====================================================
```

- output will also be in the Azure AI Foundry
- Check the UI and see you can see the evaluation output

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenteval-2.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenteval-1.jpg 'RagChat')

- You can see the evaluation output in the Azure AI Foundry
- Done