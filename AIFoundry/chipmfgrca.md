# Semiconductor Manufacturing - Root Cause Analysis - Multi Agent using Agent Framework

## Introduction

- This concept demonstrates how to leverage multiple AI agents to perform root cause analysis (RCA) in semiconductor manufacturing.
- The system uses a combination of specialized agents to analyze data, identify potential causes of defects, and recommend corrective actions.
- Have 2 agents one to do the RCA and another to create a report of the findings.
- We are using ISO 9001 Report structure for the RCA report.
- The agents collaborate to gather data, analyze it, and generate a comprehensive report.
- RCA agent will pull sample data as tools from a data source (simulated here) and perform analysis.
- The Report agent will take the findings from the RCA agent and format them into a structured report.
- This is for development and demonstration purposes only.
- We are using Microsoft Agent Framework to orchestrate the agents.
- We have also connected to Azure AI Foundry for Agent lifecycle management and monitoring. 

## Prerequisites

- Azure Subscription
- Azure AI Foundry
- Deploy gpt-4.1 chat model
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

```
import asyncio
from contextlib import AsyncExitStack
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

from agent_framework import ChatAgent, ChatMessage, ai_function
from agent_framework import AgentProtocol, AgentThread, HostedMCPTool, HostedFileSearchTool, HostedVectorStoreContent
from agent_framework.azure import AzureAIAgentClient
from azure.ai.projects.aio import AIProjectClient
from azure.identity.aio import AzureCliCredential, DefaultAzureCredential
from opentelemetry.trace import SpanKind
from opentelemetry.trace.span import format_trace_id
from agent_framework.observability import get_tracer, setup_observability
from pydantic import Field
from agent_framework import AgentRunUpdateEvent, WorkflowBuilder, WorkflowOutputEvent, SequentialBuilder
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

@ai_function
def load_chiprca_csv_files() -> str:
    tools = ""
    data_folder = Path("./data")
    for csv_file in data_folder.glob("chiprca_*.csv"):
        with open(csv_file, "r", encoding="utf-8") as f:
            content = f.read()
            tools += f"Contents of {csv_file.name}:\n{content}\n\n"
    return tools
    
async def multi_agent_interaction(query: str) -> str:
    returntxt = ""
    stack = AsyncExitStack()
    
    try:
        deployment = "gpt-5-chat-2"
        
        # Properly manage async context with AsyncExitStack
        credential = await stack.enter_async_context(AzureCliCredential())
        project_client = await stack.enter_async_context(
            AIProjectClient(
                endpoint=os.environ["AZURE_AI_PROJECT"], 
                credential=credential
            )
        )
        
        async with (
            AzureCliCredential() as credential,
            AzureAIAgentClient(async_credential=credential, 
                            project_client=project_client,
                            model_deployment_name=deployment
                            ) as chat_client,
        ):
        
            await chat_client.setup_azure_ai_observability()

            with get_tracer().start_as_current_span("ChipRCA", kind=SpanKind.CLIENT) as current_span:
                print(f"Trace ID: {format_trace_id(current_span.get_span_context().trace_id)}")
                
                # Create RCA agent - Remove 'name' parameter, use client instead
                rca_agent = chat_client.create_agent(
                    name="ChipRCAAgent",
                    instructions="You are a root cause analysis expert. Use the tools to answer the query.",
                    # Don't use 'name' parameter with ChatAgent
                    tools=[load_chiprca_csv_files],
                )

                # Create ISO9001 agent - use chat_client.create_agent for second agent
                iso9001_agent = chat_client.create_agent(
                    name="ISO9001Agent",
                    instructions="""Use your knowledge of ISO 9001 standards to ensure quality management practices are followed.
                    Make sure to reference relevant ISO 9001 clauses when providing recommendations.
                    Report should be in ISO 9001 format.
                    """,
                )
                
                workflow = (
                    WorkflowBuilder()
                    .set_start_executor(rca_agent)
                    .add_edge(rca_agent, iso9001_agent)
                    .build()
                )

                # Stream events from the workflow
                last_executor_id: str | None = None
                
                try:
                    events = workflow.run_stream(query)
                    async for event in events:
                        if isinstance(event, AgentRunUpdateEvent):
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
                            returntxt = event.data
                except Exception as e:
                    print(f"Error during workflow execution: {e}")
                    returntxt = f"Workflow error: {str(e)}"

    except Exception as e:
        print(f"Error during multi-agent interaction: {e}")
        returntxt = f"Error during multi-agent interaction: {str(e)}"
    finally:
        # Ensure all async resources are properly closed
        await stack.aclose()
        # Give time for cleanup on Windows
        await asyncio.sleep(0.25)
    
    return returntxt

if __name__ == "__main__":
    #asyncio.run(create_agents())
    # query = "Create me a catchy phrase for humanoid enabling better remote work."
    query = "Create a RCA analysis on the issues and create a ISO report."
    try:
        returntxt = asyncio.run(multi_agent_interaction(query))
        print("output:", returntxt)
    except Exception as e:
        print(f"Error during multi-agent interaction: {e}")
        pass
```

- Save the file as `chipmfgrca.py`.
- Create a folder named `data` in the same directory as the script.
- Add sample CSV files named `chiprca_1.csv`, `chiprca_2.csv`, etc., with simulated semiconductor manufacturing data for analysis.
- Run the script using:

```
python chipmfgrca.py
```

- Check the console output for the RCA analysis and ISO 9001 report.
- Sample output will include the trace ID, agent interactions, and the final report.

```
## **ISO 9001 Root Cause Analysis (RCA) Report**

**Document No.:** QMR-RCA-001  
**Date:** [Insert Date]  
**Process/Department:** [Insert Department Name]
**Reported by:** [Insert Name]  
**Reviewed by:** [Insert Auditor/Manager Name]

---

### **1. Problem Description**  
**ISO 9001 Clause Reference:** 10.2 – *Nonconformity and Corrective Action*  

Describe the issue clearly and factually (what happened, where, when, who identified it).  
**Example:**  
Customer complaints increased by 25% in Q2 due to delayed product dispatches.

---

### **2. Containment/Immediate Action**  
**ISO 9001 Clause Reference:** 8.7 – *Control of Nonconforming Outputs*  

List immediate corrective measures taken to stop the problem from worsening.  
**Example:**
- Orders were prioritized for on-time delivery.  
- Customer communication initiated to address delays.

---

### **3. Root Cause Analysis (RCA)**  
**ISO 9001 Clause Reference:** 10.2.1(b) – Determine the causes of the nonconformity  

**Method Used:** 5 Whys / Fishbone Diagram  

| Why # | Root Cause Question | Response |
|-------|---------------------|-----------|
| 1 | Why were deliveries delayed? | Insufficient manpower during peak season. |
| 2 | Why was manpower insufficient? | Hiring plan not aligned with forecasted demand. |
| 3 | Why was the hiring plan not aligned? | Demand forecasting inaccuracies and lack of review. |
| 4 | Why were forecasts not reviewed? | Absence of cross-functional review meetings. |
| 5 | Why no review meetings? | Ineffectively implemented planning procedure (QMS gap). |
```

- Now go to Azure AI Foundry portal to monitor the agents and their interactions.
- Check the Tracing section for detailed logs and trace IDs.

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/chipmfgrca-1.jpg 'RagChat')

- now for details of Agent interactions

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/chipmfgrca-2.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/chipmfgrca-3.jpg 'RagChat')

## Conclusion

- This concept demonstrates how to leverage multiple AI agents to perform root cause analysis in semiconductor manufacturing.
- The agents collaborate to analyze data and generate a structured report following ISO 9001 standards.
- This approach can be extended to include more specialized agents or integrate with real-time data sources for enhanced analysis.
- The use of Microsoft Agent Framework and Azure AI Foundry provides a scalable and manageable way to orchestrate complex multi-agent workflows.
- This is a foundational example that can be built upon for more advanced applications in industrial AI solutions.