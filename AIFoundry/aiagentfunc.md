# Azure AI Foundry Agents as HTTP Triggered Azure Function - REST API

## Introduction

- This document showcases how to create Azure AI Foundry agents and expose them as Azure Functions as HTTP triggered REST APIs.
- The idea is to create a serverless architecture where the Azure AI Foundry agents can be invoked via HTTP requests.
- The Azure Functions will enable integration with other services and applications seamlessly.
- The function repository is available at [balakreshnan/foundryagentrestapi](https://github.com/balakreshnan/foundryagentrestapi)
- This is just a sample code to get started with Azure AI Foundry agents as HTTP triggered Azure Functions.
- Not intended for production use.
- Use this as a reference to build your own serverless architecture with Azure AI Foundry agents.

## Prerequisites

- Azure AI Functions
- Azure AI Foundry
- Azure OpenAI Service
- Azure Subscription
- Azure AI Foundry Agent SDK
- Azure Functions Core Tools
- Visual Studio Code or any other code editor

## Steps

- Open Visual Studio Code and create a new Azure Functions project.
- You can use http trigger for the function.
- We will change the trigger to generic trigger later.
- we are using experimental feature of Azure functions to enable MCP.
- Here is the requirements.txt file for the Azure Functions project:

```
azure-functions
azure-ai-projects
azure-ai-agents
azure-identity
python-dotenv
openai
```

- Now creae a agent.py file to call the Azure AI Foundry Agent SDK.
- The codes creates a MCP tool that can be used to call the Azure AI Foundry Agent.
- code uses agent.py to call the agent and return the result.

- Here is the code for azure ai foundry agent.
- I am using a simple agent and function calling to demonstrate the concept.

- Here is the code for the agent.py file:

```
import asyncio
from datetime import datetime
import time
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
import os, json
from typing import Any, Callable, Set, Dict, List, Optional
from azure.ai.agents.models import CodeInterpreterTool, FunctionTool, ToolSet
from openai import AzureOpenAI

from azure.ai.agents.models import ConnectedAgentTool, MessageRole
from azure.ai.agents.models import AzureAISearchTool, AzureAISearchQueryType, MessageRole, ListSortOrder, ToolDefinition, FilePurpose, FileSearchTool
from azure.ai.agents.models import MessageTextContent, ListSortOrder

from dotenv import load_dotenv

load_dotenv()

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

def ai_default_agent(query: str) -> str:
    returntxt = ""

    # Define user functions
    user_functions = {fetch_weather}

    # Initialize the FunctionTool with user-defined functions
    functions = FunctionTool(functions=user_functions)

    # Create an agent with the Bing Grounding tool
    agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],  # Model deployment name
        name="functest-agent",  # Name of the agent
        instructions="You are a helpful agent, use the tools to answer.",  # Instructions for the agent
        # tools=code_interpreter.definitions,  # Attach the tool
        tools=functions.definitions,
    )
    print(f"Created agent, ID: {agent.id}")

    # Create a thread for communication
    thread = project_client.agents.threads.create()
    print(f"Created thread, ID: {thread.id}")
    
    # Add a message to the thread
    message = project_client.agents.messages.create(
        thread_id=thread.id,
        role="user",  # Role of the message sender
        content=query # "What is the weather in Seattle today?",  # Message content
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
        returntxt += f"Source: {message.content[0].text.value}\n"
    
    # Delete the agent when done
    project_client.agents.delete_agent(agent.id)
    project_client.agents.threads.delete(thread.id)
    print("Deleted agent and thread")

    return returntxt
```

- Now we can change the function_app.py file to call the agent.py file.
- add this to the function_app.py file:

```
@app.route(route="getdataagent")
def getdataagent(req: func.HttpRequest) -> func.HttpResponse:
    logging.info('Python HTTP trigger function processed a request.')

    query = req.params.get('query')
    if not query:
        try:
            req_body = req.get_json()
        except ValueError:
            pass
        else:
            query = req_body.get('query')

    if query:
        datars = agents.ai_default_agent(query)
        print(f"Data returned from agent: {datars}")
        return func.HttpResponse(f"Hello, {datars}. This HTTP triggered function executed successfully.")
    else:
        return func.HttpResponse(
             "This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response.",
             status_code=200
        )
```

- Basically the code will return the result from the agent.
- then send back to the client.
- Save the file and then deploy the function to Azure.
- Check the Azure portal to see if the function is deployed successfully.
- Now test the function using Postman or any other HTTP client.