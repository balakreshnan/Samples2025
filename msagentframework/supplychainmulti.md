# Microsoft Agent Framework - Supply Chain Multi-Agent Example with Azure AI Foundry integration for Agentic AI Life Cycle Management

## Introduction

- We are going to Create a Supply Chain Multi-Agent System using Microsoft Agent Framework.
- This example demonstrates a supply chain multi-agent system using Microsoft Agent Framework.
- The system integrates with Azure AI Foundry to manage supply chain tasks.
- Agents collaborate to handle procurement, inventory management, and logistics.
- Idea here is to use Azure AI Foundry as Agentic AI life Cycle management system.
- Using Azure AI Foundry as Life Cycle management system for agents.
- Ability to manage and operate agents in a production environment.
- Ability to evaluate agent performance.
- Ability to evaluate LLM and Red Teaming for agentic ai applications.

## Prerequisites

- Azure Subscription
- Azure AI Foundry
- Deploy gpt-5 chat model
- python 3.13 or above
- Install required python packages
- We need to install Azure AI Foundry Standard agent.
- We need the AI Foundry project URL and Key from Azure portal.
- Or use the default credential from Azure CLI
- Make sure permissions are set correctly for the AI Foundry project.
- Azure AI Foundry will be the Main Agentic AI life Cycle management system.
  
```
pip install msagentframework
pip install openai
pip install python-dotenv
```

## Code

- Create a utils.py file and add the following code

```python
import asyncio
import os
import json
from datetime import datetime
from random import randint
from typing import Annotated, Dict, List, Any
import requests
from pydantic import Field
from datetime import datetime, timedelta
import yfinance as yf

from dotenv import load_dotenv

# Load environment variables
load_dotenv()

def get_weather(
    location: Annotated[str, Field(description="The location to get the weather for.")],
) -> str:
    """Get the weather for a given location."""
    try:
        # Step 1: Convert location -> lat/lon using Open-Meteo Geocoding
        geo_url = f"https://geocoding-api.open-meteo.com/v1/search?name={location}&count=1"
        geo_response = requests.get(geo_url)

        if geo_response.status_code != 200:
            return f"Failed to get coordinates. Status code: {geo_response.status_code}"

        geo_data = geo_response.json()
        if "results" not in geo_data or len(geo_data["results"]) == 0:
            return f"Could not find location: {location}"

        lat = geo_data["results"][0]["latitude"]
        lon = geo_data["results"][0]["longitude"]

        # Step 2: Get weather for that lat/lon
        url = f"https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&current_weather=true"
        response = requests.get(url)

        if response.status_code == 200:
            data = response.json()
            current_weather = data.get("current_weather", {})
            temperature = current_weather.get("temperature")
            windspeed = current_weather.get("windspeed")
            return f"Weather in {location}: {temperature}°C, Wind speed: {windspeed} km/h"
        else:
            return f"Failed to get weather data. Status code: {response.status_code}"

    except Exception as e:
        import traceback
        return f"Error fetching weather data: {str(e)}\n{traceback.format_exc()}"
    
def get_ticker(company_name):
    """
    Searches for the stock ticker symbol based on the company name using Yahoo Finance search API.
    """
    url = f"https://query1.finance.yahoo.com/v1/finance/search?q={company_name}&quotesCount=1&newsCount=0"
    headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"}
    response = requests.get(url, headers=headers)
    if response.status_code != 200:
        return None
    data = json.loads(response.text)
    if data.get('quotes'):
        return data['quotes'][0]['symbol']
    return None

def fetch_stock_data(company_name) -> str:
    """
    Fetches and prints stock data for the past 7 days based on company name.
    """
    ticker = get_ticker(company_name)
    if not ticker:
        print(f"Could not find ticker for company: {company_name}")
        return
    
    # Fetch data for the past 7 days
    data = yf.download(ticker, period='7d')
    
    if data.empty:
        print(f"No data found for ticker: {ticker}")
    else:
        print(f"Stock data for {company_name} ({ticker}) over the past 7 days:")
        print(data)

    return data.to_string()
```

- Now to the main code
- We are going to create new personas for supply chain management
- Each persona will be a agent responsible for a specific task in the supply chain process
  - supplychainmonitoragent
  - disruptionpredictionagent
  - scenariosimulationagent
  - inventoryoptimizationagent
- Each above agent will have its own set of responsibilities and will work together to optimize the supply chain process.
- Each agent can use tools to get realtime or historical context to provide to agent to make decisions.
- Create a file named supplychainmulti.py and add the following code

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
import streamlit as st
from utils import get_weather, fetch_stock_data

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

from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Add Microsoft Learn MCP tool
mcplearn = HostedMCPTool(
        name="Microsoft Learn MCP",
        url="https://learn.microsoft.com/api/mcp",
        approval_mode='never_require'
    )

hfmcp = HostedMCPTool(
        name="HuggingFace MCP",
        url="https://huggingface.co/mcp",
        # approval_mode='always_require'  # for testing approval flow
        approval_mode='never_require'
    )

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

def get_chat_response_gpt5_response(query: str) -> str:
    returntxt = ""

    responseclient = AzureOpenAI(
        base_url = os.getenv("AZURE_OPENAI_ENDPOINT") + "/openai/v1/",  
        api_key= os.getenv("AZURE_OPENAI_KEY"),
        api_version="preview"
        )
    deployment = "gpt-5"

    prompt = """You are a brainstorming assistant. Given the following query, 
    provide creative ideas and suggestions.
    Here are the agents available to you:
    - supplychainmonitoragent: Monitors supply chain data and identifies anomalies.
    - disruptionpredictionagent: Predicts potential supply chain disruptions.   
    - scenariosimulationagent: Simulates alternative sourcing scenarios.
    - inventoryoptimizationagent: Optimizes inventory levels across facilities.

    Based on the query, pick the most relevant agents to assist you in brainstorming.
    Query: {query}
    
    Respond only the agents to use as array of strings.
    """

    # Some new parameters!  
    response = responseclient.responses.create(
        input=prompt.format(query=query),
        model=deployment,
        reasoning={
            "effort": "medium",
            "summary": "auto" # auto, concise, or detailed 
        },
        text={
            "verbosity": "low" # New with GPT-5 models
        }
    )

    # # Token usage details
    usage = normalize_token_usage(response.usage)

    # print("--------------------------------")
    # print("Output:")
    # print(output_text)
    returntxt = response.output_text

    return returntxt, usage

async def multi_agent_interaction(query: str) -> str:
    async with (
        AzureCliCredential() as credential,
        AzureAIAgentClient(async_credential=credential) as chat_client,
    ):
        
        supplychainmonitoragent = chat_client.create_agent(
            name="supplychainmonitoragent",
            instructions="""Continuously collect and analyze data from various tiers of the supply chain. 
        Identify anomalies and report status updates.
            """,
            #tools=... # tools to help the agent get stock prices
        )

        disruptionpredictionagent = chat_client.create_agent(
            name="disruptionpredictionagent",
            instructions="""Use historical and real-time data to forecast potential disruptions. 
        Provide risk scores and mitigation suggestions.
            """,
            #tools=... # tools to help the agent get stock prices
        )

        scenariosimulationagent = chat_client.create_agent(
            name="scenariosimulationagent",
            instructions="""Generate and evaluate alternative sourcing scenarios based on current supply chain data and predicted disruptions.
            """,
            #tools=... # tools to help the agent get stock prices
        )

        inventoryoptimizationagent = chat_client.create_agent(
            name="inventoryoptimizationagent",
            instructions="""Analyze inventory levels across global facilities and recommend adjustments to maintain resilience and efficiency.
            """,
            #tools=... # tools to help the agent get stock prices
        )
        workflow = (
            WorkflowBuilder()
            #.add_agent(ideation_agent, id="ideation_agent")
            #.add_agent(inquiry_agent, id="inquiry_agent", output_response=True)
            .set_start_executor(supplychainmonitoragent)
            .add_edge(supplychainmonitoragent, disruptionpredictionagent)
            .add_edge(disruptionpredictionagent, scenariosimulationagent)
            .add_edge(scenariosimulationagent, inventoryoptimizationagent)
            # .add_multi_selection_edge_group(supplychainmonitoragent, 
            #                                 [disruptionpredictionagent, 
            #                                  scenariosimulationagent, 
            #                                  inventoryoptimizationagent], 
            #                                 selection_func=get_chat_response_gpt5_response(query=query))  # Example of multi-selection edge
            .build()
        )

        # Stream events from the workflow. We aggregate partial token updates per executor for readable output.
        last_executor_id: str | None = None

        events = workflow.run_stream(query)
        async for event in events:
            if isinstance(event, AgentRunUpdateEvent):
                # AgentRunUpdateEvent contains incremental text deltas from the underlying agent.
                # Print a prefix when the executor changes, then append updates on the same line.
                eid = event.executor_id
                if eid != last_executor_id:
                    if last_executor_id is not None:
                        print()
                    print(f"{eid}:", end=" ", flush=True)
                    last_executor_id = eid
                print(event.data, end="", flush=True)
            elif isinstance(event, WorkflowOutputEvent):
                print("\n===== Final output =====")
                print(event.data)

        #chat_client.delete_agent(ideation_agent.id)
        #chat_client.delete_agent(inquiry_agent.id)
        # await chat_client.project_client.agents.delete_agent(ideation_agent.id)
        # await chat_client.project_client.agents.delete_agent(inquiry_agent.id)
        # chat_client.project_client.agents.threads.delete(ideation_agent.id)

    return "Done"

if __name__ == "__main__":
    #asyncio.run(create_agents())
    # query = "Create me a catchy phrase for humanoid enabling better remote work."
    query = "Create a AI Data center to handle 1GW capacity with 10MW power usage effectiveness.With 250000 GPU's."
    asyncio.run(multi_agent_interaction(query))
```

- Now save the file and run the code using the command below

```
python supplychainmulti.py
```

- here is the query used in the example
  - "Create a AI Data center to handle 1GW capacity with 10MW power usage effectiveness.With 250000 GPU's."
- Ask questions and see the output from each agent in the supply chain multi-agent system.