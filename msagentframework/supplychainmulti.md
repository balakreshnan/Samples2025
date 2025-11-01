# Microsoft Agent Framework - Supply Chain Multi-Agent Example with Azure AI Foundry integration for Agentic AI Life Cycle Management

![info](https://github.com/balakreshnan/Samples2025/blob/main/msagentframework/images/SupplyChainDesigner.png 'RagChat')

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
- here is the output

```
supplychainmonitoragent: Certainly! Below is a **high-level architecture and design outline** for an AI data center capable of handling **1 GW (1000 MW) overall capacity**, with a **Power Usage Effectiveness (PUE) of 1.1**, accommodating **250,000 GPUs**, and actual IT load of **10 MW** (from your question, I interpret 10MW as the IT load per data hall, not total load, but will clarify both):

---

## 1. **Summary Table**

| Attribute                  | Value              |
|----------------------------|--------------------|
| Total Data Center Power    | 1 GW (1000 MW)     |
| PUE                       | 1.1                |
| IT Load                   | 900 MW             |
| Facility Overhead          | 100 MW             |
| Number of GPUs             | 250,000            |
| Typical GPU Power (NVIDIA H100) | 400 W         |
| Total GPU Power Required   | 100 MW (GPUs only) |
| Remaining IT Power (Servers/Storage) | 800 MW    |

*(Assumption: Each GPU is ~400W load. Actual model may vary.)*

## 2. **Capacity Calculation**

- **GPU Load:**
  250,000 GPUs × 400 W = **100,000,000 W = 100 MW**
- **Non-GPU Servers & Storage:**
  Remaining IT load = 900 MW – 100 MW = **800 MW**
- **Facility Overhead (Cooling, Security, etc.):**
  1000 MW × (1.1 – 1) = **100 MW**

---

## 3. **High-Level Data Center Layout**

#### **a. Location & Site Facilities**
- Choose location with robust grid access, renewable energy integration, water access for cooling (if needed), and robust fiber connectivity.
- Example: Split into **10 Data Center Halls**, each with **100 MW** (~10,000 GPUs per hall).

#### **b. Power Architecture**
- **Substations:** Main substation(s) providing 1 GW.
- **UPS System:** Tier IV or V architecture. N+2 redundancy.
- **Power Distribution:**
  - Distributed via switchgear → busways → PDUs.
  - **Rack Power:** Each rack supports 10–40 kW (with advanced cooling).

#### **c. Cooling Infrastructure**
- **Liquid Cooling:** Direct-to-chip, rear-door heat exchangers, immersion cooling.
- **Chillers/Cooling Towers:** Sufficient for extreme density.
- **Redundant HVAC loops and thermal energy storage.**

#### **d. GPU Clusters**
- **Rack Design:**
  - If each rack houses ~40 GPUs: 250,000 / 40 = **6,250 Racks** for GPUs.
  - Each rack ~16 kW (for GPU alone).
  - IT racks for storage, networking additional.

#### **e. Networking**
- **High-Bandwidth Fabric:** 400/800 Gbps Ethernet or Infiniband.
- **Multiple Redundant Fiber Paths.**
- **Distributed aggregation switches, core routers, SDN for AI workloads.**

#### **f. Storage**
- Petabyte-scale, NVMe flash, distributed SAN/NAS.
- AI-optimized storage tiering.

#### **g. Management & Security**
- DCIM Software for power, thermal, asset tracking.
- Physical security: Biometric, video, perimeter alarms.
- Cybersecurity: Zero-trust, isolation, continuous monitoring.

---

## 4. **Sustainability**
- **Power Source:** Renewable (solar/wind/hydro) mix, on-site generation, battery backup, grid agreements.
- Advanced heat recovery for district heating.

---

## 5. **Reference Schematic**

```
┌───────────────────────────┐
│   Main Substation (1GW)   │
├─────────┬─────────────────┴┬─────────────┐
│UPS/Generators│  Switchgear │ Cooling Plant│
├─────────┴───┬─────┬────────┴───┬─────────┤
│   Data Hall 1 │...│   Data Hall 10        │
│  (10,000 GPUs) │   │  (10,000 GPUs)      │
│   IT Racks    │   │   IT Racks           │
│ Networking    │   │   Networking         │
└─────┬───────────────┴───────────────┬────┘
      │      DCIM / Security / NOC    │
      └───────────────────────────────┘
```

---

## 6. **Implementation Considerations**

- **Space:** 6,250+ IT racks, plus overhead for networking/storage/aux.
- **Phased Build:** Can be staged in ~10×100MW blocks (~⅒ scaling).
- **Advanced Automation:** AI/ML for DC management, predictive maintenance.
- **Regulatory Compliance:** Data privacy, cross-border data flow, safety.

---

## 7. **Sample Bill of Materials (BoM)**

- **GPUs:** 250,000×NVIDIA H100
- **Servers:** High-density, multi-GPU servers.
- **Racks:** 6,250+ advanced cooling racks.
- **Networking:** 400/800 Gbps switches, fiber cabling.
- **UPS:** 1 GW total, N+2 config.
- **Chillers:** Sufficient for ~100 MW thermal per hall.
- **Management:** DCIM/Security appliances.
- **Storage:** PB-scale NVMe arrays.

---

## 8. **Cost Estimate**
- **CapEx:** $5–8 Billion USD (2024 estimate for 1GW Campus)
- **OpEx:** $100–200 Million/year (energy, staffing, maintenance)

---

# **Summary Diagram**

[If you upload a site sketch, I can annotate/expand or review for you.]

---

## **Let me know if you want a more detailed breakdown for:**

- Cooling choices (liquid, immersion, rear-door HX)
- Network architecture (leaf-spine, AI fabrics)
- Capacity scaling (modular buildout)
- Sustainability strategies (solar farms, battery backup)
- Vendor comparison (NVIDIA vs AMD vs others)

You can **upload a site plan** or request **schematic diagrams** for specific components!

---

**Would you like a detailed rack allocation plan, or expansion into operational workflows for AI workloads?**
disruptionpredictionagent: Let's break down what it takes to design an AI data center handling **1GW (1,000MW) total capacity**, targeting **10MW power usage effectiveness per module**, with **250,000 GPUs**. Below is a conceptual blueprint covering power, GPU deployment, cooling, networking, and facility:

---

## 1. **Power & Capacity Planning**

- **Total Capacity:** 1GW (entire campus)
- **Module/Pod Design:** 10MW per pod for ease of control, redundancy, and modular growth (100 pods for 1GW).
- **Power Usage Effectiveness (PUE):** Aim for industry-leading PUE (e.g., 1.1), implying 900MW for IT, 100MW for cooling, lighting, etc.

---

## 2. **GPU Deployment**

- **Total GPUs:** 250,000 units
- **Typical GPU Power:** Modern AI GPUs (NVIDIA H100, MI300X) ≈ 400W each
- **Total GPU Power Load:** 250,000 × 0.4kW = **100MW**
- **Racks Needed:** Each rack can host 40 GPUs (@ 16kW/rack with liquid cooling)
  - 250,000 / 40 = 6,250 racks
- **Modularity:** Distribute GPUs across the 10MW modules (2,500 GPUs per module)

---

## 3. **Data Center Facility**

### **a. Floorplan & Infrastructure**
- **Power:** Each 10MW pod has dedicated substation, generators, switchgear, UPS
- **Cooling:** 
  - Direct-to-chip liquid cooling, rear-door chillers, or full immersion cooling
  - Redundant chilled water loops and heat exchangers
- **Space:** 
  - Each pod: ~250 racks (with GPU and supporting servers)
  - Total facility floor area: Estimated 250,000 – 400,000 sqft

### **b. Networking**
- **Interconnects:** High-speed fabric (Infiniband, 400/800Gbps Ethernet)
- **Core Routers & Switches:** Redundant, with multiple uplinks to external IXPs/transit
- **Physical Paths:** Fiber and copper, per pod and cross-campus

### **c. Storage and Compute**
- **NVMe Flash Storage:** Petascale arrays for training datasets, distributed in pods
- **Clustered AI Servers:** Multi-GPU per node, with supporting CPUs for orchestration

---

## 4. **Energy Source & Sustainability**

- **Grid Connection:** Robust 1GW feed, with on-site transformers and switchgear
- **Backup:** N+2 redundancy, diesel/gas generators, battery arrays
- **Renewables:** Solar/wind integration, in line with green targets

---

## 5. **Schematic Structure**

```
AI DATA CENTER CAMPUS (1GW)

┌────────────Campus────────────┐
│ 100 x 10MW Data Center Pods  │
│ (each pod: 2,500 GPUs, racks)│
│ ---------------------------- │
│ Power (substation, UPS)      │
│ Cooling (liquid/immersion)   │
│ Networking (fiber fabric)    │
│ Storage (flash arrays)       │
│ Management/Security          │
└──────────────────────────────┘
```

---

## 6. **High-Level Bill of Materials**

- **250,000 x AI GPUs**
- **6,250 x High-density racks (liquid cooled)**
- **Network switches/routers (>=400Gbps per rack)**
- **Power switchgear, UPS, backup generators**
- **Chillers, liquid cooling infrastructure**
- **Security (physical/cyber), DCIM management software**
- **Onsite/adjacent renewable power generation (as required)**

---

## 7. **Risk & Mitigation**

- **Risk:** Power/cooling failure  
   **Mitigation:** N+2 redundancy, modular isolation, predictive maintenance
- **Risk:** Network congestion  
   **Mitigation:** High-density mesh, redundant uplinks, AI-orchestrated traffic
- **Risk:** Fire/flood  
   **Mitigation:** VESDA smoke detectors, sealed immersion, advanced fire suppression
- **Risk:** Capacity oversubscription
   **Mitigation:** Dynamic load allocation, hot/cold aisle containment, phased expansion

---

## 8. **Scalability & Expansion**

- Design in 10MW modular pods for easy scaling up/down
- Supports future GPU technologies (denser, higher-wattage, etc)
- Upgradeable networking and cooling infrastructure

---

## 9. **Summary**

This campus will be one of the world’s largest, designed for high-density AI workloads and optimized for sustainability and reliability. 

If you share the desired site plan, I'll help with the schematic illustration!  
Would you like a **3D block diagram, layout drawing, or detailed module design** next? Please specify!
scenariosimulationagent: Here is an outline for a high-capacity AI data center designed for 1GW total power, targeting 10MW PUE segments, hosting 250,000 GPUs:

---

### 1. **Power and GPU Deployment Calculation**

- **Total Power Budget (1GW):** 1,000MW (entire data center campus)
- **PUE Target:** Let's assume 1.1 (modern AI data centers, aggressive but feasible)
  - **IT (Compute) Power:** 1,000MW / 1.1 ≈ **910MW**
  - **Facility Power (cooling, uninterruptible power supply, etc.):** ≈ 90MW

- **GPU Count:** 250,000
- **Typical GPU Power Consumption:** ~400W per GPU (NVIDIA H100-class)
  - **Total GPU Power Draw:** 250,000 x 0.4kW = **100MW**
  - **Total Compute Power (incl. CPUs, storage, etc):** Balance of ~810MW

- **Rack Calculation (if 40 GPUs/rack):** 250,000 / 40 = **6,250 racks**
- **Rack Power Density:** 40 GPUs x 400W = **16kW per rack** (will need liquid/immersion cooling)

---

### 2. **Data Center Architecture**

**a. Modular "Pod" Design:**
- Divide facility into 10MW segments (pods), ~100 pods for full scale.
- Each pod: ~2,500 GPUs (~63 racks), independent power/cooling/network zones.

**b. Power Infrastructure:**
- On-site substation(s) supporting high voltage grid connection, ideally green/renewable energy as much as possible.
- N+2 UPS and generator redundancy.
- Busways and rack power distribution for high-density compute.
- Independent metering per pod for capacity and cost management.

**c. Cooling:**
- Liquid cooling for racks (direct-to-chip, rear-door heat exchangers, or immersion cooling).
- Redundant chillers, cooling towers, heat exchange loops.
- Scalable to high rack densities (16kW+ per rack).

**d. Networking:**
- Fabric switches with 400/800Gbps Ethernet or InfiniBand.
- Highly redundant edge/core, multiple carriers for wide-area connectivity.
- Air-gapped and segmented by pod for workload isolation.

**e. Storage:**
- Petabyte-scale NVMe flash arrays per pod for training datasets.
- Tiered cold storage locations (lower power draw, densified at campus perimeter).

---

### 3. **Environmental and Security**

- Onsite battery arrays and diesel/natural gas backup for blackouts.
- Smart fire suppression (VESDA), air monitoring, water leak sensors.
- Advanced cyber defense (physical separation for sensitive AI workloads).
- Sustainable building (rainwater recovery, solar/wind integration, heat re-use).

---

### 4. **Sample Floor Plan Diagram**

```
CAMPUS SUBSTATION
       │
       ├── Pod 1 (10MW) ──[2500 GPU]
       ├── Pod 2 (10MW) ──[2500 GPU]
       └── ...
       └── Pod 100 (10MW)─[2500 GPU]
       │
    DCIM / Network Core / Security Hub
```

---

### 5. **Sample Build Materials**

- **250,000 GPUs** (e.g., NVIDIA H100 or AMD Instinct)
- **6,250 Liquid-cooled racks**
- **100 Modular cooling units/plant**
- **100 UPS/generator systems**
- **100 Pod network cores (400-800Gbps)**
- **DCIM software and sensors**
- **Physical security infrastructure**

---

### 6. **Scalability & Management**

- Scalable by pod/module: easy phased ramp-up as demand grows.
- AI-enabled data center management (DCIM): predictive maintenance, load balancing, cooling automation.

---

### 7. **Cost Estimate (2024, Approximate)**

- CapEx (campus): **$6-10B** (site, GPUs, construction, cooling, power, network)
- OpEx/year: **$100-250M** (power, staff, maintenance)

---

## **Summary**

This approach delivers extreme scale and density (1GW, 250k GPUs), with modular isolation, optimal power/cooling, future scalability, and high network transport. If you have site drawings, I can help with more detailed planning (zoning, load, flow). Let me know if you want further breakdowns (rack layouts, cooling systems, compute/storage mix, cost curve, etc.).
inventoryoptimizationagent: Here's a high-level design outline for an **AI Data Center with 1GW (1,000MW) total power**, targeting **10MW per module** and hosting **250,000 GPUs**.

---

## 1. **Requirements Recap**

- **Total Data Center Facility Power:** 1,000MW
- **PUE Target:** 1.1 or lower (modern best-practice)
- **Number of GPUs:** 250,000
  - Typical GPU (NVIDIA H100): 400W load
  - **Total GPU Power Draw:** 250,000 × 0.4kW = **100MW** for GPUs
- **Modularization:** 10MW modules (pods/halls) for manageability and resilience

---

## 2. **Facility Design**

**A. Modular Pod/Block Layout**
- 1,000MW / 10MW per module = **100 modules/pods**
- Each pod contains 2,500 GPUs (2,500 × 0.4kW = 1MW for GPUs alone)
- Remaining 9MW per pod for CPUs, storage, switches, and other IT/infra

**B. Power Distribution**
- Redundant substations and switchgear
- N+2 UPS & GenSets per module
- Busways delivering 10MW to each pod
- Metering at pod and rack level for transparency and optimization

**C. Cooling**
- Direct-to-chip liquid cooling or full immersion for GPU racks
- Redundant chiller plants or free-cooling with adiabatic support
- High-density containment (hot aisle/cold aisle or liquid isolators)

**D. Racking & Floor Space**
- At 40 GPUs/rack, 2,500 GPUs/pod = **63 racks/pod** × 100 pods = **~6,300 racks**
- Each rack ~16kW+ (requires liquid cooling, no air-cooling at this density)
- Inter-pod space for networking, power, fire suppression, service corridors

**E. Networking**
- At least 400G–800G backbone, Infiniband or similar low-latency fabric
- Redundant border routers per module, multiple external fiber providers
- Multiple isolated zones for resilience

**F. Storage**
- NVMe flash/AI-optimized storage clusters per pod, aggregated at site level
- 10+ PB capacity with expansion for AI training data

---

## 3. **Operations & Resilience**

- **Active Racks Monitoring:** Power, temp, GPU, and cooling sensor telemetry; automated or AI-driven load shifting
- **Security:** Physical separation, 2FA/Biometrics, CCTV, intrusion detection, mantraps
- **Fire Suppression:** VESDA, inert-gas or clean-agent, dry-pipe water system
- **Sustainability:** Green grid, on-site battery/solar/wind as adjunct, heat re-use systems

---

## 4. **Scalability & Modularity**

- Pods can be staged/commissioned as demand grows
- Spare modules for redundancy, N+2 topology for major systems
- Zoning allows for failure isolation, quick swaps, and expansion

---

## 5. **Sample Block Diagram**

```
[GRID/Utility]-->[Substation]-->[100 x 10MW Modules]--\
                                                    |--[Aggregate Core Network]--[Border/IXP]
   [Chillers/Free Coolers]----/
```

Each 10MW module contains:
- 63 GPU racks, supporting (liquid-cooled)
- Server/network/storage racks as needed
- Pod-level switches, KVM, sensors

---

## 6. **Staffing/Automation**

- AI-powered DCIM system for monitoring and control
- 24/7 NOC/SOC
- Robotics for physical movement/maintenance (optional at huge scale)

---

## 7. **Estimated CapEx**

- **Data center ($10M/MW):** $10B (very rough modern ballpark--location and spec dependent)
- **GPUs (at $25k/unit):** $6.25B
- **Networking/storage/infrastructure:** $1–3B

---

**This design is resilient, AI/ML and hyperscale-oriented, and maximizes performance and density.** 

If you need more detail (e.g., sample rack elevations, power paths, cooling water flow, or single-pod blueprints), **let me know!**
===== Final output =====
Here's a high-level design outline for an **AI Data Center with 1GW (1,000MW) total power**, targeting **10MW per module** and hosting **250,000 GPUs**.

---

## 1. **Requirements Recap**

- **Total Data Center Facility Power:** 1,000MW
- **PUE Target:** 1.1 or lower (modern best-practice)
- **Number of GPUs:** 250,000
  - Typical GPU (NVIDIA H100): 400W load
  - **Total GPU Power Draw:** 250,000 × 0.4kW = **100MW** for GPUs
- **Modularization:** 10MW modules (pods/halls) for manageability and resilience

---

## 2. **Facility Design**

**A. Modular Pod/Block Layout**
- 1,000MW / 10MW per module = **100 modules/pods**
- Each pod contains 2,500 GPUs (2,500 × 0.4kW = 1MW for GPUs alone)
- Remaining 9MW per pod for CPUs, storage, switches, and other IT/infra

**B. Power Distribution**
- Redundant substations and switchgear
- N+2 UPS & GenSets per module
- Busways delivering 10MW to each pod
- Metering at pod and rack level for transparency and optimization

**C. Cooling**
- Direct-to-chip liquid cooling or full immersion for GPU racks
- Redundant chiller plants or free-cooling with adiabatic support
- High-density containment (hot aisle/cold aisle or liquid isolators)

**D. Racking & Floor Space**
- At 40 GPUs/rack, 2,500 GPUs/pod = **63 racks/pod** × 100 pods = **~6,300 racks**
- Each rack ~16kW+ (requires liquid cooling, no air-cooling at this density)
- Inter-pod space for networking, power, fire suppression, service corridors

**E. Networking**
- At least 400G–800G backbone, Infiniband or similar low-latency fabric
- Redundant border routers per module, multiple external fiber providers
- Multiple isolated zones for resilience

**F. Storage**
- NVMe flash/AI-optimized storage clusters per pod, aggregated at site level
- 10+ PB capacity with expansion for AI training data

---

## 3. **Operations & Resilience**

- **Active Racks Monitoring:** Power, temp, GPU, and cooling sensor telemetry; automated or AI-driven load shifting
- **Security:** Physical separation, 2FA/Biometrics, CCTV, intrusion detection, mantraps
- **Fire Suppression:** VESDA, inert-gas or clean-agent, dry-pipe water system
- **Sustainability:** Green grid, on-site battery/solar/wind as adjunct, heat re-use systems

---

## 4. **Scalability & Modularity**

- Pods can be staged/commissioned as demand grows
- Spare modules for redundancy, N+2 topology for major systems
- Zoning allows for failure isolation, quick swaps, and expansion

---

## 5. **Sample Block Diagram**

```
[GRID/Utility]-->[Substation]-->[100 x 10MW Modules]--\
                                                    |--[Aggregate Core Network]--[Border/IXP]
   [Chillers/Free Coolers]----/
```

Each 10MW module contains:
- 63 GPU racks, supporting (liquid-cooled)
- Server/network/storage racks as needed
- Pod-level switches, KVM, sensors

---

## 6. **Staffing/Automation**

- AI-powered DCIM system for monitoring and control
- 24/7 NOC/SOC
- Robotics for physical movement/maintenance (optional at huge scale)

---

## 7. **Estimated CapEx**

- **Data center ($10M/MW):** $10B (very rough modern ballpark--location and spec dependent)
- **GPUs (at $25k/unit):** $6.25B
- **Networking/storage/infrastructure:** $1–3B

---

**This design is resilient, AI/ML and hyperscale-oriented, and maximizes performance and density.**

If you need more detail (e.g., sample rack elevations, power paths, cooling water flow, or single-pod blueprints), **let me know!**
```

- Done! You have successfully created a multi-agent supply chain management system using Azure OpenAI and Python. Each agent has its own responsibilities and they work together to optimize the supply chain process.