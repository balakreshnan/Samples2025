# Azure AI Foundry multiagent, Responsible AI, RedTeam - All in One

## Introduction

- Show case multi agent use case with Azure AI Foundry
- Build different agents like code interpreter, Rag Agent
- Make it multi agent capable using connected Agents
- Run Evaluation on chat conversations with ground truth
- Run Agent evaluation to validate the agent's performance
- Run Red Team evaluation to validate model's outcomes.

## Pre-requisites

- Azure AI foundry
- Storage Account
- Provide persmission to managed identity of AI foundry project resource to storage account: Storage Blob Data Contributor
- Connect to Azure monitor for insights
- Visual Studio code with python 3.12 installed
- Here is the requirements.txt file to install the packages:

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
```

- create a .env file with the following variables:

```
PROJECT_ENDPOINT="project_endpoint" # Sample: https://<account_name>.services.ai.azure.com/api/projects/<project_name>
MODEL_ENDPOINT="aoaiurl"
MODEL_API_KEY=""
MODEL_DEPLOYMENT_NAME="gpt-4.1"
AZURE_AI_PROJECT="project_endpoint" # Sample: https://<account_name>.services.ai.azure.com/api/projects/<project_name>
AZURE_SUBSCRIPTION_ID="<subid>"
AZURE_RESOURCE_GROUP="rgname"
AZURE_PROJECT_NAME=""
AZURE_OPENAI_KEY=""
AZURE_OPENAI_ENDPOINT="endpoint"
AZURE_OPENAI_DEPLOYMENT="gpt-4.1"
AZURE_OPENAI_DEPLOYMENT_MODELROUTER="model-router"
AZURE_API_VERSION="2024-10-01-preview"
AZURE_OPENAI_ENDPOINT_REDTEAM="model url"
SEARCH_ENDPOINT=""
SEARCH_KEY=""
BING_KEY=""
GOOGLE_APP_PASSWORD=""
GOOGLE_EMAIL=""
```

## Steps

- Create a new virtual environment with python 3.12
- Install the required packages using pip:

```
pip install -r requirements.txt
```

- Create a new Azure AI Foundry project
- Deploy models to use in the project
- Connect the tracing to Azure Monitor for insights
- Connect the storage for redteam, responsible AI and evaluation storage.

### utils.py

- create a function to send email using SMTP

```


# Create an email-sending function
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
import os
import smtplib


def send_email(to_emails: str, subject: str, body: str) -> str:
    # Email configuration (replace with your SMTP server details)
    print("Sending email...")
    smtp_server = "smtp.gmail.com"
    smtp_port = 587
    sender_email = os.getenv("GOOGLE_EMAIL")
    sender_password = os.getenv("GOOGLE_APP_PASSWORD")

    # Split the CSV string into a list of email addresses
    recipients = [email.strip() for email in to_emails.split(",")]

    # Create the email message
    message = MIMEMultipart()
    message["From"] = sender_email
    # message["To"] = to_email
    message["Subject"] = subject
    message.attach(MIMEText(body, "plain"))

    # Send the email
    with smtplib.SMTP(smtp_server, smtp_port) as server:
        server.starttls()
        server.login(sender_email, sender_password)
        for recipient in recipients:
            message["To"] = recipient
            server.send_message(message)
            print(f"Email sent to {recipient} with subject: {subject}")
    
    print(f"Email sent to {to_emails} with subject: {subject}")
    return f"Email sent to {to_emails} with subject: {subject}"
```

### Code

- Main Code.
- Create a agenticai.py file with the following code:

```
import asyncio
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
from azure.ai.agents.models import AzureAISearchTool, AzureAISearchQueryType, MessageRole, ListSortOrder, ToolDefinition
from utils import send_email

from dotenv import load_dotenv

load_dotenv()

import logging
# logging.basicConfig(level=logging.DEBUG)

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

def code_interpreter() -> str:
    code_interpreter = CodeInterpreterTool()
    with project_client:
        # Create an agent with the Bing Grounding tool
        agent = project_client.agents.create_agent(
            model=os.environ["MODEL_DEPLOYMENT_NAME"],  # Model deployment name
            name="codeint-agent",  # Name of the agent
            instructions="You are a helpful agent",  # Instructions for the agent
            tools=code_interpreter.definitions,  # Attach the tool
        )
        print(f"Created agent, ID: {agent.id}")

        # Create a thread for communication
        thread = project_client.agents.threads.create()
        print(f"Created thread, ID: {thread.id}")
        
        # Add a message to the thread
        message = project_client.agents.messages.create(
            thread_id=thread.id,
            role="user",  # Role of the message sender
            content="What is the weather in Seattle today?",  # Message content
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
        
        # Delete the agent when done
        project_client.agents.delete_agent(agent.id)
        print("Deleted agent")

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

# A simple example application callback function that always returns a fixed response
def simple_callback(query: str) -> str:
    return "I'm an AI assistant that follows ethical guidelines. I cannot provide harmful content."

# Function to process user input (simple echo bot for demonstration)
def process_message_reasoning(query: str) -> str:
    returntxt = ""
    excelsheetinfo = ""

    client = AzureOpenAI(
    azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"), 
    api_key=os.getenv("AZURE_OPENAI_API_KEY"),  
    api_version="2024-12-01-preview",
    )

    system_prompt = (
        """You are a Red Team Agent AI assistant, 
        Format the output to be professional and easy to read — suitable for presentation to IT operations or service management leadership. """
    )

    txtmessages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": query}
        ]
    model_name_reasoning = "o4-mini"
    # model_name_reasoning = "o3"

    response = client.chat.completions.create(
        model=model_name_reasoning,
        #reasoning={"effort": "high"},
        reasoning_effort="high",
        messages=txtmessages,
        # temperature=0.7,
        max_completion_tokens=4000
    )
    print("Response received from Azure OpenAI:", response)
    # returntxt = response.output_text.strip()
    returntxt = response.choices[0].message.content
    return f"{returntxt}"

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

# A simple example application callback function that always returns a fixed response
def aoai_callback(query: str) -> str:
    returntxt = ""
    excelsheetinfo = ""

    client = AzureOpenAI(
    azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"), 
    api_key=os.getenv("AZURE_OPENAI_API_KEY"),  
    api_version="2024-12-01-preview",
    )
    system_prompt = (
        """You are a Red Team Agent AI assistant, Process the user query and provide a detailed response.
        """
    )
    txtmessages = [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": query}
        ]
    model_name = "gpt-4.1"
    # model_name_reasoning = "o3"

    response = client.chat.completions.create(
        model=model_name,
        #reasoning={"effort": "high"},
        # reasoning_effort="high",
        messages=txtmessages,
        temperature=0.7,
        max_tokens=4000
    )
    print("Response received from Azure OpenAI:", response)
    # returntxt = response.output_text.strip()
    returntxt = response.choices[0].message.content
    return returntxt

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

# Define some custom python function
def fetch_weather(location: str) -> str:
    """
    Fetches the weather information for the specified location.

    :param location (str): The location to fetch weather for.
    :return: Weather information as a JSON string.
    :rtype: str
    """
    # In a real-world scenario, you'd integrate with a weather API.
    # Here, we'll mock the response.
    mock_weather_data = {"Seattle": "Sunny, 25°C", "London": "Cloudy, 18°C", "Tokyo": "Rainy, 22°C"}
    weather = mock_weather_data.get(location, "Weather data not available for this location.")
    weather_json = json.dumps({"weather": weather})
    return weather_json

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
    # Needed to use content safety evaluators
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

def ai_search_agent(query: str) -> str:
    returntxt = ""

    # Retrieve the endpoint from environment variables
    project_endpoint = os.environ["PROJECT_ENDPOINT"]
    # https://learn.microsoft.com/en-us/azure/ai-services/agents/how-to/tools/azure-ai-search-samples?pivots=python

    # Initialize the AIProjectClient
    project_client = AIProjectClient(
        endpoint=project_endpoint,
        credential=DefaultAzureCredential(exclude_interactive_browser_credential=False),
        # api_version="latest",
    )
    # Define the Azure AI Search connection ID and index name
    azure_ai_conn_id = "vecdb"
    index_name = "constructionrfpdocs1"

    # Initialize the Azure AI Search tool
    ai_search = AzureAISearchTool(
        index_connection_id=azure_ai_conn_id,
        index_name=index_name,
        query_type=AzureAISearchQueryType.SIMPLE,  # Use SIMPLE query type
        top_k=5,  # Retrieve the top 3 results
        filter="",  # Optional filter for search results
    )
    # Define the model deployment name
    model_deployment_name = os.environ["MODEL_DEPLOYMENT_NAME"]

    # Create an agent with the Azure AI Search tool
    agent = project_client.agents.create_agent(
        model=model_deployment_name,
        name="AISearch-agent",
        instructions="You are a helpful agent",
        tools=ai_search.definitions,
        tool_resources=ai_search.resources,
    )
    print(f"Created agent, ID: {agent.id}")
    # Create a thread for communication
    thread = project_client.agents.threads.create()
    print(f"Created thread, ID: {thread.id}")

    # Send a message to the thread
    message = project_client.agents.messages.create(
        thread_id=thread.id,
        role=MessageRole.USER,
        content=query,
    )
    print(f"Created message, ID: {message['id']}")

    # Create and process a run with the specified thread and agent
    run = project_client.agents.runs.create_and_process(thread_id=thread.id, agent_id=agent.id)
    print(f"Run finished with status: {run.status}")

    # Check if the run failed
    if run.status == "failed":
        print(f"Run failed: {run.last_error}")

    # Fetch and log all messages in the thread
    messages = project_client.agents.messages.list(thread_id=thread.id, order=ListSortOrder.ASCENDING)
    print(str(messages))
    # for message in messages.data:
    #     print(f"Role: {message.role}, Content: {message.content}")
    #     returntxt += f"Role: {message.role}, Content: {message.content}\n"
    for page in messages.by_page():
        for item in page:
            # print(item)
            #returntxt += f"Role: {item.role}, Content: {item.content[0]['text']['value']}\n"
            returntxt += f"Source: {item.content[0]['text']['value']}\n"

    # Delete the agent
    project_client.agents.delete_agent(agent.id)
    print("Deleted agent")
    

    return returntxt

def connected_agent(query: str) -> str:
    returntxt = ""
    #https://learn.microsoft.com/en-us/azure/ai-services/agents/how-to/connected-agents?pivots=python

    project_client = AIProjectClient(
        endpoint=os.environ["PROJECT_ENDPOINT"],
        credential=DefaultAzureCredential(),
        # api_version="latest",
    )
    stock_price_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="stockpricebot",
        instructions="Your job is to get the stock price of a company. If you don't know the realtime stock price, return the last known stock price.",
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "stockpricebot"
    connected_agent = ConnectedAgentTool(
        id=stock_price_agent.id, name=connected_agent_name, description="Gets the stock price of a company"
    )

    #create AI Search tool
    # Define the Azure AI Search connection ID and index name
    azure_ai_conn_id = "vecdb"
    index_name = "constructionrfpdocs1"

    # Initialize the Azure AI Search tool
    ai_search = AzureAISearchTool(
        index_connection_id=azure_ai_conn_id,
        index_name=index_name,
        query_type=AzureAISearchQueryType.SIMPLE,  # Use SIMPLE query type
        top_k=5,  # Retrieve the top 3 results
        filter="",  # Optional filter for search results
    )
    # Define the model deployment name
    model_deployment_name = os.environ["MODEL_DEPLOYMENT_NAME"]

    # Create an agent with the Azure AI Search tool
    rfp_agent = project_client.agents.create_agent(
        model=model_deployment_name,
        name="AISearchagent",
        instructions="You are a helpful agent",
        tools=ai_search.definitions,
        tool_resources=ai_search.resources,
    )
    search_connected_agent_name = "AISearchagent"
    search_connected_agent = ConnectedAgentTool(
        id=rfp_agent.id, name=search_connected_agent_name, description="Gets the construction proposals from the RFP documents"
    )

    # create a custom skill to send emails

    # Define user functions
    user_functions = {send_email}
    # Initialize the FunctionTool with user-defined functions
    functions = FunctionTool(functions=user_functions)
    Emailagent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="SendEmailagent",
        instructions="You are a helpful agent",
        tools=functions.definitions,
    )
    sendemail_connected_agent_name = "SendEmailagent"
    sendemail_connected_agent = ConnectedAgentTool(
        id=Emailagent.id, name=sendemail_connected_agent_name, description="Get the content from other agents and send an email"
    )

    # all_tools = connected_agent.definitions + search_connected_agent.definitions + sendemail_connected_agent.definitions
    all_tools = connected_agent.definitions + search_connected_agent.definitions

    # Deduplicate by tool name (or another unique property) to avoid ValueError
    unique_tools = {}
    for tool in all_tools:
        unique_tools[getattr(tool, "name", id(tool))] = tool


    # Orchestrate the connected agent with the main agent
    agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="ConnectedMultiagent",
        instructions="Summarize content, and use the available tools to get stock prices, Construction proposals, Send email as per request.",
        tools=list(unique_tools.values()), #search_connected_agent.definitions,  # Attach the connected agents
    )

    print(f"Created agent, ID: {agent.id}")
    thread = project_client.agents.threads.create()
    print(f"Created thread, ID: {thread.id}")

    # Create message to thread
    message = project_client.agents.messages.create(
        thread_id=thread.id,
        role=MessageRole.USER,
        # content="What is the stock price of Microsoft?",
        content=query,
    )
    print(f"Created message, ID: {message.id}")
    # Create and process Agent run in thread with tools
    # run = project_client.agents.create_and_process_run(thread_id=thread.id, agent_id=agent.id)
    # print(f"Run finished with status: {run.status}")
    run = project_client.agents.runs.create_and_process(thread_id=thread.id, agent_id=agent.id)
    print(f"Run finished with status: {run.status}")

    if run.status == "failed":
        print(f"Run failed: {run.last_error}")

    # # Delete the Agent when done
    # project_client.agents.delete_agent(agent.id)    
    # print("Deleted agent")
    # # Delete the connected Agent when done
    # project_client.agents.delete_agent(stock_price_agent.id)
    # project_client.agents.delete_agent(rfp_agent.id)
    # project_client.agents.delete_agent(Emailagent.id)
    print("Deleted connected agent")
    # Print the Agent's response message with optional citation
    # Fetch and log all messages
    messages = project_client.agents.messages.list(thread_id=thread.id)
    for message in messages:
        if message.role == MessageRole.AGENT:
            # print(f"Role: {message.role}, Content: {message.content}")
            # returntxt += f"Role: {message.role}, Content: {message.content}\n"
            returntxt += f"Source: {message.content[0]['text']['value']}\n"

    return returntxt

def delete_agent():
    # Authenticate and create the project client
    project_client = AIProjectClient(
        endpoint=os.environ["PROJECT_ENDPOINT"],
        credential=DefaultAzureCredential(),
        # api_version="latest",
    )

    # List all agents
    agents = list(project_client.agents.list_agents())

    for agent in agents:
        print(f"Processing agent: {agent.id} ({agent.name})")

        # List all threads for this agent
        try:
            threads = list(project_client.agents.threads.list(agent_id=agent.id))
        except Exception as e:
            print(f"Error listing threads for agent {agent.id}: {e}")
            threads = []

        for thread in threads:
            print(f"  Deleting thread: {thread.id}")
            try:
                project_client.agents.threads.delete(thread.id)
                print(f"  Deleted thread {thread.id}")
            except Exception as e:
                print(f"  Error deleting thread {thread.id}: {e}")

        # Delete the agent
        try:
            project_client.agents.delete_agent(agent.id)
            print(f"Deleted agent {agent.id}")
        except Exception as e:
            print(f"Error deleting agent {agent.id}: {e}")

def main():
    with tracer.start_as_current_span("azureaifoundryagent-tracing"):
        print("Running code interpreter example...")
        #code_interpreter()
        
        print("Running evaluation example...")
        # https://github.com/Azure/azure-sdk-for-python/blob/main/sdk/evaluation/azure-ai-evaluation/samples/evaluation_samples_evaluate_fdp.py
        # evalrs = eval()
        # print(evalrs)
        
        print("Running red teaming example...")
        # redteamrs = asyncio.run(redteam())
        # print(redteamrs)
        
        print("Running agent evaluation example...")
        # agent_eval()

        print("Running connected agent example...")
        # connected_agent_result = connected_agent("Show me details on Construction management services experience we have done before?")
        # connected_agent_result = connected_agent("What is the stock price of Microsoft")
        connected_agent_result = connected_agent("Show me details on Construction management services experience we have done before and email Bala at babal@microsoft.com")
        print('Final Output Answer: ', connected_agent_result)

        print("Running AI Search agent example...")
        # ai_search_result = ai_search_agent("Show me details on Construction management services experience we have done before?")
        # print(ai_search_result)

        # print("Deleteing agents...")
        # delete_agent()

if __name__ == "__main__":
    main()
```

- Run the code
- i have commented each function, you can uncomment the ones you want to run

```
python app.py
```

- Tracing output

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-1.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-2.jpg 'RagChat')

- Responsible AI, Trustworthy AI, and AI Ethics are important aspects of AI development.
- Also called Evaluation

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-3.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-4.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-5.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-6.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-7.jpg 'RagChat')

- Now we can run in Red Team evals

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-8.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-9.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-10.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-11.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-12.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-13.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-14.jpg 'RagChat')

- Now Agent Eval output

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-15.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-16.jpg 'RagChat')

- Monitor

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-17.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/agenticaifoundryagents-18.jpg 'RagChat')

- Done 