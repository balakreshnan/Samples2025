# Enabling Azure AI Agent as MCP using Azure Functions

## Prerequisites

- Azure AI Functions
- Azure AI Foundry
- Azure OpenAI Service
- Azure Subscription
- Azure AI Foundry Agent SDK

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

- now we are going to create a function to call the agent.py file and return the result.
- name is function_app.py replace the below content.

```
import json
import azure.functions as func
import logging
import agents

app = func.FunctionApp(http_auth_level=func.AuthLevel.ANONYMOUS)

# Constants for the Azure Blob Storage container, file, and blob path
_SNIPPET_NAME_PROPERTY_NAME = "snippetname"
_SNIPPET_PROPERTY_NAME = "snippet"
_BLOB_PATH = "snippets/{mcptoolargs." + _SNIPPET_NAME_PROPERTY_NAME + "}.json"


class ToolProperty:
    def __init__(self, property_name: str, property_type: str, description: str):
        self.propertyName = property_name
        self.propertyType = property_type
        self.description = description

    def to_dict(self):
        return {
            "propertyName": self.propertyName,
            "propertyType": self.propertyType,
            "description": self.description,
        }


# Define the tool properties using the ToolProperty class
tool_properties_save_snippets_object = [
    ToolProperty(_SNIPPET_NAME_PROPERTY_NAME, "string", "The name of the snippet."),
    ToolProperty(_SNIPPET_PROPERTY_NAME, "string", "The content of the snippet."),
]

tool_properties_get_snippets_object = [ToolProperty(_SNIPPET_NAME_PROPERTY_NAME, "string", "The name of the snippet.")]

# Convert the tool properties to JSON
tool_properties_save_snippets_json = json.dumps([prop.to_dict() for prop in tool_properties_save_snippets_object])
tool_properties_get_snippets_json = json.dumps([prop.to_dict() for prop in tool_properties_get_snippets_object])

@app.generic_trigger(
    arg_name="context",
    type="mcpToolTrigger",
    toolName="get_aiagentdata",
    description="Retrieve data from ai agents.",    
    toolProperties=json.dumps([ {"propertyName": "query", "propertyType": "string", "description": "Whats the weather in Seattle today?"} ]),
)
def get_aiagentdata(context) -> str:
    """
    Retrieves a snippet by name from Azure Blob Storage.

    Args:
        file (func.InputStream): The input binding to read the snippet from Azure Blob Storage.
        context: The trigger context containing the input arguments.

    Returns:
        str: The content of the snippet or an error message.
    """
    # snippet_content = file.read().decode("utf-8")
    content = json.loads(context)
    query_from_args = content["arguments"].get("query", "") 
    datars = agents.ai_default_agent(query_from_args)
    print(f"Data returned from agent: {datars}")
    logging.info(f"Retrieved snippet: {datars}")
    return f"Hello, {datars}. This HTTP triggered function executed successfully."
```

- Now use visual studio code and deploy the Azure Functions project to Azure.
- Go to Azure Portal and in function portal blade, get the key for the function.
- Here is the URL format

```
https://<your-function-app>.azurewebsites.net/runtime/webhooks/mcp/sse?code=<system_key>
```

## Client code to test

- here is the code to validate if the function is working as expected.
- Code loads the function url for MCP from environment variable `MCP_FUNCTION_URL`.
- Because it has key in the URL

```
import json
import requests
import streamlit as st
import streamlit.components.v1 as components
from openai import AzureOpenAI, OpenAIError

import base64
import os
from gtts import gTTS
import tempfile
import uuid
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Azure OpenAI configuration (replace with your credentials)
AZURE_ENDPOINT = os.getenv("AZURE_OPENAI_ENDPOINT")
AZURE_API_KEY = os.getenv("AZURE_OPENAI_KEY")
WHISPER_DEPLOYMENT_NAME = "whisper"
CHAT_DEPLOYMENT_NAME = os.getenv("AZURE_OPENAI_DEPLOYMENT")

# Initialize Azure OpenAI client
client = AzureOpenAI(
    azure_endpoint=AZURE_ENDPOINT,
    api_key=AZURE_API_KEY,
    api_version="2024-06-01"  # Adjust API version as needed
)

def mcp_generate_chat_response(context):
    """Generate a chat response using Azure OpenAI with tool calls."""
    returntxt = ""

    prompt = f"""
    You are a helpful assistant. Use the following context and tools to answer the user's query.
    If the context or tools are not relevant, provide a general response based on the query.
    Only respond with the tool call.
    Ask for followup until you get the right information. Probe the user for more details if necessary.
    If the context is not relevant, provide a general response based on the query.
    Be positive and encouraging in your response. Ignore any negative or irrelevant information.
    please ignore any questions that are not related to learning. 
    DOn't get annoyed or frustrated. if user asks probing questions, please politely ignore them.
    Provide sources and citations for your responses.
    Can you make the output more conversational so that a text to speech model can read it out loud it more practical way.

    User Query:
    {context}
    
    Response:
    """
    messages = [
        {"role": "system", "content": "You are a helpful assistant with access to the MCP API."},
        {"role": "user", "content": prompt}
    ]

    mcpclient = AzureOpenAI(  
        base_url = os.getenv("AZURE_OPENAI_ENDPOINT") + "/openai/v1/",  
        api_key= os.getenv("AZURE_OPENAI_KEY"),
        api_version="preview"
        )

    response = mcpclient.responses.create(
        model=CHAT_DEPLOYMENT_NAME, # replace with your model deployment name 
        tools=[
            {
                "type": "mcp",
                "server_label": "aiagentmcp",
                "server_url": os.getenv("MCP_FUNCTION_URL"),
                "require_approval": "never"
            },
        ],
        input=context,
        max_output_tokens= 1500,
        instructions="Generate a response using the MCP API tool.",
    )
    # returntxt = response.choices[0].message.content.strip()
    retturntxt = response.output_text    
    print(f"Response: {retturntxt}")
        
    return retturntxt, None

def process_main():

    context = "what is the weather in Seattle today?"
    response, _ = mcp_generate_chat_response(context)
    print(f"Chatbot response: {response}")

if __name__ == "__main__":
    process_main()
```

- now save the file as `mcpclient.py` and run the file using the following command.

```
python mcpclient.py
```

- wait for the function to respond. You should see the response from the Azure AI Foundry Agent in the console.