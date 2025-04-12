# Azure AI Foundry Agents - Building Knowledge Agents as APIs

## Introduction

- Create Azure Functions to run the AI Agents using the Azure AI Foundry SDK.
- This document provides a sample using Code Spaces and Github Repository to create the Azure Functions.
- Everything is created in cloud, saved in Cloud repository and deployed and run in cloud.
- Using python SDK to create the Azure Functions.
- Running in Consumption plan.
- Using Azure AI Search to create the Knowledge base.
- Using Bing Grounding to create the Web Knowledge base.
- These are specific technology specific agents which can be repurposed to add any indsutry specific knowledge to ground the AI Agents.
- Bing is more web content so we cannot change.
- But AI Search agents is custom documents uploaded to the Azure AI Search service.

## Prerequisites

- Azure Subscription
- Azure AI Foundry Resource created.
- Azure AI Search Resource created.
- Azure Bing Grounding Resource created.
- Azure Functions created.
- Using the Azure Open AI Service inside the Azure AI Foundry.
- Github Account created.
- Create a sample repository in Github.
- Go to Code spaces and then create new code space.
- Select the repository created in the previous step.

## Code

- First create the environment file like below.

```
AZURE_OPENAI_API_KEY=""
AZURE_OPENAI_KEY=""
AZURE_OPENAI_ENDPOINT=""
AZURE_OPENAI_DEPLOYMENT=""
AZURE_OPENAI_API_VERSION=""
AZURE_AI_SEARCH_ENDPOINT=""
AZURE_AI_SEARCH_ENDPOINT_NAME=""
AZURE_AI_SEARCH_KEY=""
AZURE_AI_SEARCH_INDEX=""
AZURE_OPENAI_RED_ENDPOINT=""
AZURE_OPENAI_RED_KEY=""
AZURE_OPENAI_RED_DEPLOYMENT=""
AZURE_SUBSCRIPTION_ID=""
AZURE_RESOURCE_GROUP=""
AZUREAI_PROJECT_NAME=""
PROJECT_CONNECTION_STRING_EASTUS2=""
GOOGLE_APP_PASSWORD=""
GOOGLE_EMAIL=""
BING_CONNECTION_NAME=""
```

- Create a requirements.txt file with the below content.

```
autogenstudio
PyPDF2
autogen-core
autogen-agentchat
autogen-ext[openai]
autogen-ext[magentic-one,openai,web-surfer]
autogen-ext
azure-ai-projects
azure-monitor-opentelemetry
opentelemetry-instrumentation-openai-v2
autogen-ext[web-surfer]
azure-ai-evaluation
python-docx
openai
python-pptx
arxiv
playwright
PyMuPDF
marshmallow==3.18.0
azure-ai-evaluation[redteam] 
azure-functions
azurefunctions-extensions-http-fastapi
python-dotenv
azure.functions
```

- First install Azure functions core tools in Visual Studio Code.
- Then go to Command Palette and select Azure Functions: Create New Project.
- Select the name, location, langauge, version and trigger type. I am using HTTP trigger.
- Select the Authorization level as Function.
- Create a sample function and give a name.
- Now you should see the defalt host.json and function.json files created.
- You will also see a function_app.py file created.
- This is where we are going to add the code for the Azure Function.
- First do the imports.

```python
import azure.functions as func
# from azurefunctions.extensions.http.fastapi import Request, StreamingResponse
import logging
import asyncio
import os
import subprocess
from datetime import datetime
from typing import List, Dict

from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import FileSearchTool, MessageAttachment, FilePurpose
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from azure.ai.projects.models import AzureAISearchTool, AzureAISearchQueryType
from azure.ai.projects.models import RequiredFunctionToolCall, SubmitToolOutputsAction, ToolOutput, ToolSet
from typing import Any, Callable, Set, Dict, List, Optional
from openai import AzureOpenAI

from dotenv import load_dotenv
load_dotenv()
```

- Load environment variables using dotenv.
- Make sure to create a .env file in the root directory and add the environment variables.
- Make sure to add the .env file to .gitignore so that it is not pushed to the repository.
- setup Azure AI Foundry Client and Azure OpenAI Client.

```python
project_client = AIProjectClient.from_connection_string(
    credential=DefaultAzureCredential(),
    conn_str=os.environ["PROJECT_CONNECTION_STRING_EASTUS2"],
)

client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"), 
  api_key=os.getenv("AZURE_OPENAI_API_KEY"),  
  api_version="2024-10-21",
)

model_name = os.getenv("AZURE_OPENAI_DEPLOYMENT")
```

- Here is the code for AI Search agent using Azure AI Foundry SDK.

```python
def aisearch_process_agent(query: str) -> str:

    project_client = AIProjectClient.from_connection_string(
        credential=DefaultAzureCredential(),
        conn_str=os.environ["PROJECT_CONNECTION_STRING_EASTUS2"],
    )

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
    ai_search = AzureAISearchTool(index_connection_id=conn_id, index_name="constrfp",
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
    # print(f"Messages: {messages}")
    # rs = parse_output(messages)
    # print("Messages: ", assistant_message)
    returnstring = assistant_message
    return returnstring

@app.route(route="aisearch")
async def aisearch(req: func.HttpRequest) -> func.HttpResponse:
    returnstring = ""
    logging.info('Python HTTP trigger function processed a request.')

    query = req.params.get('query')
    if not query:
        query = "what are the best practicse from Virgnia Railway express project?"
    returnstring = aisearch_process_agent(query=query)

    return func.HttpResponse(f"AI Search Results Summarized: {returnstring}.")
```

- Now here is the code for Bing Grounding agent using Azure AI Foundry SDK.

```python
def bingground_process_agent(query: str) -> str:

    project_client = AIProjectClient.from_connection_string(
        credential=DefaultAzureCredential(),
        conn_str=os.environ["PROJECT_CONNECTION_STRING_EASTUS2"],
    )
    
    bing_connection = project_client.connections.get(
        connection_name=os.environ["BING_CONNECTION_NAME"]
    )
    conn_id = bing_connection.id
    print(conn_id)

    returnmessage = ""

    # Initialize agent bing tool and add the connection id
    bing = BingGroundingTool(connection_id=conn_id)

    # Create agent with the bing tool and process assistant run
    with project_client:
        agent = project_client.agents.create_agent(
            model="gpt-4o",
            name="BingGroundAssistant",
            instructions="You are a helpful assistant",
            tools=bing.definitions,
            headers={"x-ms-enable-preview": "true"}
        )
        print(f"Created agent, ID: {agent.id}")

        # Create thread for communication
        thread = project_client.agents.create_thread()
        print(f"Created thread, ID: {thread.id}")

        # Create message to thread
        message = project_client.agents.create_message(
            thread_id=thread.id,
            role="user",
            content="What is the top news today",
        )
        print(f"Created message, ID: {message.id}")

        # Create and process agent run in thread with tools
        run = project_client.agents.create_and_process_run(thread_id=thread.id, agent_id=agent.id)
        print(f"Run finished with status: {run.status}")

        # Retrieve run step details to get Bing Search query link
        # To render the webpage, we recommend you replace the endpoint of Bing search query URLs with `www.bing.com` and your Bing search query URL would look like "https://www.bing.com/search?q={search query}"
        run_steps = project_client.agents.list_run_steps(run_id=run.id, thread_id=thread.id)
        run_steps_data = run_steps['data']
        print(f"Last run step detail: {run_steps_data}")

        if run.status == "failed":
            print(f"Run failed: {run.last_error}")

        # Delete the assistant when done
        project_client.agents.delete_agent(agent.id)
        print("Deleted agent")

        # Fetch and log all messages
        messages = project_client.agents.list_messages(thread_id=thread.id)
        print(f"Messages: {messages}")
        returnmessage = messages

    return returnmessage

@app.route(route="bingsearch")
async def bingsearch(req: func.HttpRequest) -> func.HttpResponse:
    returnstring = ""
    logging.info('Python HTTP trigger function processed a request.')

    query = req.params.get('query')
    if not query:
        query = "Show me top 5 latest news about Microsoft."
    returnstring = bingground_process_agent(query=query)

    return func.HttpResponse(f"Bing Search Results Summarized: {returnstring}.")
```

- Go to Command Palette and select Azure Functions: Deploy to Function App.
- if no function app is created, it will prompt to create a new one.
- Deploy the function app and it will take a while to deploy.
- Once deployed, you will see the function app in the Azure portal.
- Output will show the URL for the function app.
- Take the URL and test the function app using Postman or any other tool.

- Here is the url: https://functionappname.azurewebsites.net/api/aisearch?query=what%20are%20the%20best%20practicse%20from%20Virgnia%20Railway%20express%20project
- Here is a sample output from the function app using AI Search. Data is open source and not real data.

```
AI Search Results Summarized: Best practices drawn from the Virginia Railway Express (VRE) project on construction management services include:

1. **Comprehensive Project Management Requirements**:
   - Require project managers to possess significant qualifications, such as a Bachelor's degree or 20 years of experience in construction and project management. Certification, such as from the Construction Management Association of America (CMAA), is also preferred【3:0†source】.

2. **Detailed Proposal and Evaluation Criteria**:
   - Establish clear evaluation criteria for assessing capabilities, expertise, and past performance of firms and subcontractors involved in the project【3:0†source】.

3. **Best Practices in Implementation**:
   - Develop and implement project control tools and procedures, ensuring timely and accurate data organization for updated reporting and tracking【3:2†source】.
   - Provide oversight for job cost budgets, schedule adherence, and project scopes, ensuring efficiency【3:2†source】.
   - Train staff on program controls and forecasting methods to enhance consistency in project execution【3:2†source】.

These approaches contribute to achieving quality, cost-efficiency, and compliance with project scopes and timelines in large-scale transportation construction projects..
```

- https://url/api/bingsearch?query=what%20are%20the%20top%2010%20latest%20business%20news for today
- Here is the sample output from the function app using Bing Grounding. Data is open source and not real data.

```
Bing Search Results Summarized: {'object': 'list', 'data': [{'id': 'msg_SIPsrHLr9RB9edMfl0i2EpTj', 'object': 'thread.message', 'created_at': 1744463682, 'assistant_id': 'asst_wfh8rFELPUJPzePjc04wOP6a', 'thread_id': 'thread_ItwUlcVAdpp0OdNjBmHwQSho', 'run_id': 'run_a6GB9G1rzYZG21F5zKbKhgcn', 'role': 'assistant', 'content': [{'type': 'text', 'text': {'value': "I found diverse topics trending today:\n\n1. Preparations for Expo 2025 in Osaka, Japan, which anticipates significant global engagement from April to October【5:1†source】.\n2. Political updates in the U.S., including critiques of former President Trump's management of economic issues【5:2†source】.\n3. Global humanitarian concerns such as the Rohingya crisis and challenges in agriculture in the Global South【5:8†source】.\n\nLet me know if you want details on any of these.", 'annotations': [{'type': 'url_citation', 'text': '【5:1†source】', 'start_index': 156, 'end_index': 168, 'url_citation': {'url': 'https://english.elpais.com/archive/2025-04-12/', 'title': 'Daily news for April 12, 2025 | EL PAÍS in English - EL PAÍS English ...'}}, {'type': 'url_citation', 'text': '【5:2†source】', 'start_index': 281, 'end_index': 293, 'url_citation': {'url': 'https://www.newsweek.com/thebulletin/2025-04-12', 'title': 'The Bulletin April 12, 2025 - Newsweek'}}, {'type': 'url_citation', 'text': '【5:8†source】', 'start_index': 404, 'end_index': 416, 'url_citation': {'url': 'https://www.globalissues.org/news/2025/04', 'title': 'News headlines in April 2025 - Global Issues'}}]}}], 'attachments': [], 'metadata': {}}, {'id': 'msg_pxeDEwS5jJO1VgA8O3DtfIRA', 'object': 'thread.message', 'created_at': 1744463678, 'assistant_id': None, 'thread_id': 'thread_ItwUlcVAdpp0OdNjBmHwQSho', 'run_id': None, 'role': 'user', 'content': [{'type': 'text', 'text': {'value': 'What is the top news today', 'annotations': []}}], 'attachments': [], 'metadata': {}}], 'first_id': 'msg_SIPsrHLr9RB9edMfl0i2EpTj', 'last_id': 'msg_pxeDEwS5jJO1VgA8O3DtfIRA', 'has_more': False}.
```

- i am using Flex consumer plan for the Azure Function.
- Haven't tried this in container yet.
- i also added ping api to valdiate if the function is running or not.
- Have fun creating endless agents apis that can cloud scale and run in cloud.