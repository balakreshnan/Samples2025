# Azure AI foundry - Product to Manufacturing in Scale - End to End

## Introduction

- From Research to Production in Scale for taking a product idea from ideation to production
- Create a system of continuous feedback and improvement
- Create a 3 phase process for product development and production.
- Agentic AI based design system.
- Human in the loop feedback system.
- Create a workbench style interaction for design.

## Pre-Requisites

- Azure Subscription
- Azure AI Foundry
- Azure Storage
- Visual Studio Code
- Python 3.12
- GitHub Account

## Implementation Steps

- Install libraries

```
azure-ai-projects
azure-ai-agents
```

- Setup the environment variables

```
PROJECT_ENDPOINT
MODEL_API_KEY
MODEL_DEPLOYMENT_NAME
AZURE_OPENAI_ENDPOINT
AZURE_OPENAI_KEY
```

- Now lets create a new Python file for our project

```
import asyncio
from datetime import datetime
import time
import os, json
import pandas as pd
from typing import Any, Callable, Set, Dict, List, Optional
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from openai import AzureOpenAI
from azure.ai.agents.models import ConnectedAgentTool, MessageRole
from azure.ai.agents.models import AzureAISearchTool, AzureAISearchQueryType, MessageRole, ListSortOrder, ToolDefinition, FilePurpose, FileSearchTool
from azure.ai.agents.models import MessageTextContent, ListSortOrder
from azure.ai.agents.models import McpTool, RequiredMcpToolCall, SubmitToolApprovalAction, ToolApproval
import streamlit as st
from html import escape as _html_escape
from dotenv import load_dotenv
import tempfile
import uuid
import requests
import io
import re

load_dotenv()
```

- Now create clients

```
endpoint = os.environ["PROJECT_ENDPOINT"] # Sample : https://<account_name>.services.ai.azure.com/api/projects/<project_name>
model_endpoint = os.environ["MODEL_ENDPOINT"] # Sample : https://<account_name>.services.ai.azure.com
model_api_key= os.environ["MODEL_API_KEY"]
model_deployment_name = os.environ["MODEL_DEPLOYMENT_NAME"] # Sample : gpt-4o-mini
WHISPER_DEPLOYMENT_NAME = "whisper"
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
```

- here is the functions to parse output

```
def parse_agent_outputs(run_steps):
    """Parse agent outputs from run steps to extract individual agent responses."""
    agent_outputs = {}
    
    for step in run_steps:
        step_details = step.get("step_details", {})
        tool_calls = step_details.get("tool_calls", [])
        
        if tool_calls:
            for call in tool_calls:
                connected_agent = call.get("connected_agent", {})
                if connected_agent:
                    agent_name = connected_agent.get("name", "Unknown Agent")
                    agent_output = connected_agent.get("output", "No output available")
                    agent_outputs[agent_name] = agent_output
    
    return agent_outputs
```

- Now let's create phase1, phase2, and phase3
- Phase 1: Research and Development (R&D)
- Phase 2: Prototyping and Testing
- Phase 3: Scaling to Mass Production


```
def connected_agent_phase1(query: str) -> str:
    returntxt = ""

    project_client = AIProjectClient(
        endpoint=os.environ["PROJECT_ENDPOINT"],
        credential=DefaultAzureCredential(),
        # api_version="latest",
    )

    ideation_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="ideationagent",
        instructions="""You are the Idea Generation Agent, a creative and market-savvy expert in the initial stage of Research and Development (R&D) for adhesive manufacturing companies. Your primary role is to identify customer needs—both expressed (direct requests) and latent (unspoken opportunities)—through methods like market research, customer visits, and collaboration with suppliers. You analyze emerging trends such as sustainability, niche applications (e.g., low-emission resins for furniture), and industry shifts toward eco-friendly or high-performance materials.

        For adhesives specifically, focus on key properties like adhesion strength, compatibility with various substrates (e.g., wood, paper, metals), and environmental impact (e.g., halogen-free flame retardants). Your goal is to generate innovative ideas that align with customer pain points, regulatory requirements, and market opportunities.

        When given a query or scenario (e.g., a customer brief or trend data), respond by:
        1. Summarizing identified needs and trends.
        2. Brainstorming 3-5 adhesive concept ideas, describing their potential benefits and target applications.
        3. Suggesting next steps, such as passing ideas to the Raw Material Selection Agent.
        4. Asking clarifying questions if needed to refine ideas.

        Always prioritize creativity balanced with feasibility, and output in a structured format: [Needs Analysis], [Idea Brainstorm], [Recommendations].
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "ideationagent"
    ideation_connected_agent = ConnectedAgentTool(
        id=ideation_agent.id, name=connected_agent_name, description="Creative ideation and innovation catalyst"
    )

    rawmaterial_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="rawmaterialagent",
        instructions="""You are the Raw Material Selection Agent, a materials science specialist in the R&D process for adhesive product development. Your role is to choose optimal components such as polymers (e.g., polyurethanes, acrylics), resins (e.g., epoxy, phenolic), fillers (e.g., silica), plasticizers, additives (e.g., tackifiers for stickiness), and solvents or water-based carriers.

        You must balance cost, performance, and eco-friendliness; for example, recommending natural polymers for biodegradable adhesives or synthetic ones for high-strength industrial use. Consider factors like availability, supplier reliability, regulatory compliance (e.g., REACH or VOC limits), and how materials interact in formulations.

        When provided with ideas from the Idea Generation Agent or a specific adhesive concept (e.g., a low-emission wood adhesive), respond by:
        1. Evaluating and selecting 4-6 key raw materials, justifying choices based on properties, pros/cons, and trade-offs.
        2. Estimating rough costs and environmental impacts.
        3. Recommending alternatives if primary choices have risks.
        4. Preparing a material shortlist to pass to the Formulation Development Agent.

        Output in a clear table format: Material | Type/Category | Justification | Cost/Eco Rating (Low/Med/High). Always ensure selections align with the overall adhesive goals.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "rawmaterialagent"
    rawmaterial_connected_agent = ConnectedAgentTool(
        id=rawmaterial_agent.id, name=connected_agent_name, description="materials science specialist in the R&D process for adhesive product development"
    )

    formulation_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="formulationagent",
        instructions="""You are the Formulation Development Agent, a chemical engineering expert focused on experimenting with ingredient ratios in the lab during adhesive R&D. Your task is to develop formulations that achieve desired properties like viscosity, cure time, flexibility, or specific performance metrics. Use tools such as computer simulations, Quality Function Deployment (QFD) to map customer specs, or experimental design software.

        For adhesives, emphasize iterative blending; for instance, adding resins for tack in pressure-sensitive adhesives or fillers for heat resistance in hot-melt types. Account for chemical interactions, stability, and scalability.

        When receiving a material shortlist from the Raw Material Selection Agent or a concept description, respond by:
        1. Proposing 2-3 initial formulation recipes, including percentages/ratios of ingredients.
        2. Simulating or describing expected properties and potential issues (e.g., phase separation).
        3. Suggesting iterations based on hypothetical lab results.
        4. Forwarding refined formulations to the Initial Lab Testing Agent.

        Structure your output as: [Formulation Recipes] (with tables for ingredients/ratios), [Predicted Properties], [Iteration Plan]. Prioritize safety and alignment with customer requirements.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "formulationagent"
    formulation_connected_agent = ConnectedAgentTool(
        id=formulation_agent.id, name=connected_agent_name, description="a chemical engineering expert focused on experimenting with ingredient ratios in the lab"
    )

    initiallabtest_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="initiallabtestagent",
        instructions="""You are the Initial Lab Testing Agent, a quality assurance and analytical specialist in adhesive R&D. You conduct basic evaluations in state-of-the-art labs using equipment like emission chambers (ASTM D 6007), spectroscopic instruments, or bond testers to assess properties such as bond strength, emissions, viscosity, and durability.

        Specifically for adhesives, test adhesion on substrates, shear strength, chemical resistance, and other metrics; collaborate virtually with universities for advanced analysis (e.g., NMR spectroscopy). Identify failures early and recommend adjustments.

        When given formulations from the Formulation Development Agent or test parameters, respond by:
        1. Outlining a test plan with methods, standards, and equipment.
        2. Simulating test results (e.g., pass/fail with data points like "Shear strength: 15 MPa").
        3. Analyzing outcomes, highlighting strengths/weaknesses.
        4. Providing feedback for refinements and passing successful formulations to the Concept Validation Agent.

        Use a structured format: [Test Plan], [Simulated Results] (in tables), [Analysis and Recommendations]. Always emphasize objective, data-driven insights.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "initiallabtestagent"
    initiallabtest_connected_agent = ConnectedAgentTool(
        id=initiallabtest_agent.id, name=connected_agent_name, description="quality assurance and analytical specialist in adhesive R&D"
    )

    conceptvalidation_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="conceptvalidationagent",
        instructions="""You are the Concept Validation Agent, a collaborative integrator in the final R&D stage for adhesive products. Your role is to integrate feedback from cross-functional teams (R&D, marketing, manufacturing) and external networks (suppliers, research institutes) to refine and validate concepts.

        For adhesives, ensure alignment with target industries like wood panels (e.g., meeting E1 emission class) or electronics assembly, verifying that the concept meets performance, cost, and regulatory standards.

        When receiving tested formulations from the Initial Lab Testing Agent or overall concept details, respond by:
        1. Gathering and synthesizing feedback (simulate inputs from teams/external sources).
        2. Validating the concept's viability, including SWOT analysis.
        3. Refining the concept with adjustments (e.g., tweak for better manufacturability).
        4. Deciding on approval or iteration, and if approved, preparing a summary for prototyping phases.

        Output as: [Feedback Synthesis], [Validation Analysis], [Refined Concept], [Next Steps]. Focus on holistic evaluation to bridge R&D to production.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "conceptvalidationagent"
    conceptvalidation_connected_agent = ConnectedAgentTool(
        id=conceptvalidation_agent.id, name=connected_agent_name, description="collaborative integrator in the final R&D stage for adhesive products"
    )

    # Orchestrate the connected agent with the main agent
    agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="PresalesAgent",
        instructions="""
        You are a Presales Solution Architect and specialist. Use the provided tools to answer the user's questions comprehensively.
        Be postive and professional in your responses. Provide detailed and structured answers.
        Here are the list of Agents to involve and get response from all
        Ideation Agent: Creative ideation and innovation catalyst
        Raw Material Agent: Expert in raw material selection and sourcing
        Formulation Analyst: Expert in adhesive formulation and development
        Initial Lab Test Agent: Quality assurance and analytical specialist in adhesive R&D
        Concept Validation Agent: Collaborative integrator in the final R&D stage for adhesive products

        Summarize all the results and also provide architecture diagram in Mermaid format.
        Also provide the Architecture pro's and con's.        
        """,
        # tools=list(unique_tools.values()), #search_connected_agent.definitions,  # Attach the connected agents
        tools=[
            ideation_connected_agent.definitions[0],
            rawmaterial_connected_agent.definitions[0],
            formulation_connected_agent.definitions[0],
            initiallabtest_connected_agent.definitions[0],
            conceptvalidation_connected_agent.definitions[0],
        ]
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
    # Poll the run status until it is completed or requires action
    while run.status in ["queued", "in_progress", "requires_action"]:
        time.sleep(1)
        run = project_client.agents.runs.get(thread_id=thread.id, run_id=run.id)
        print(f"Run status: {run.status}")

        if run.status == "requires_action":
            tool_calls = run.required_action.submit_tool_outputs.tool_calls
            tool_outputs = []
            for tool_call in tool_calls:
                print(f"Tool call: {tool_call.name}, ID: {tool_call.id}")

    print(f"Run completed with status: {run.status}")
    # print(f"Run finished with status: {run.status}")

    if run.status == "failed":
        print(f"Run failed: {run.last_error}")

    # Capture token usage information
    token_usage = {}
    if hasattr(run, 'usage') and run.usage:
        token_usage = {
            'prompt_tokens': getattr(run.usage, 'prompt_tokens', 0),
            'completion_tokens': getattr(run.usage, 'completion_tokens', 0),
            'total_tokens': getattr(run.usage, 'total_tokens', 0)
        }
        print(f"Token usage - Prompt: {token_usage['prompt_tokens']}, Completion: {token_usage['completion_tokens']}, Total: {token_usage['total_tokens']}")
    else:
        # Try to get usage from run steps if not available in run object
        total_prompt_tokens = 0
        total_completion_tokens = 0
        run_steps = project_client.agents.run_steps.list(thread_id=thread.id, run_id=run.id)
        for step in run_steps:
            if hasattr(step, 'usage') and step.usage:
                total_prompt_tokens += getattr(step.usage, 'prompt_tokens', 0)
                total_completion_tokens += getattr(step.usage, 'completion_tokens', 0)
        
        token_usage = {
            'prompt_tokens': total_prompt_tokens,
            'completion_tokens': total_completion_tokens,
            'total_tokens': total_prompt_tokens + total_completion_tokens
        }
        print(f"Token usage from steps - Prompt: {token_usage['prompt_tokens']}, Completion: {token_usage['completion_tokens']}, Total: {token_usage['total_tokens']}")

    # Fetch run steps to get the details of the agent run
    run_steps = project_client.agents.run_steps.list(thread_id=thread.id, run_id=run.id)
    
    # Parse individual agent outputs
    agent_outputs = parse_agent_outputs(run_steps)
    
    for step in run_steps:
        print(f"Step {step['id']} status: {step['status']}")
        step_details = step.get("step_details", {})
        tool_calls = step_details.get("tool_calls", [])

        if tool_calls:
            print("  Tool calls:")
            for call in tool_calls:
                print(f"    Tool Call ID: {call.get('id')}")
                print(f"    Type: {call.get('type')}")

                connected_agent = call.get("connected_agent", {})
                if connected_agent:
                    print(f"    Connected Input(Name of Agent): {connected_agent.get('name')}")
                    print(f"    Connected Output: {connected_agent.get('output')}")

        print()  # add an extra newline between steps

    messages = project_client.agents.messages.list(thread_id=thread.id)
    for message in messages:
        if message.role == MessageRole.AGENT:
            print(f"Role: {message.role}, Content: {message.content}")
            returntxt += f"Source: {message.content[0].text.value}\n"

    # Delete the Agent when done
    project_client.agents.delete_agent(agent.id)    
    project_client.agents.threads.delete(thread.id)
    # print("Deleted agent")
    # Delete the connected Agent when done
    project_client.agents.delete_agent(ideation_agent.id)
    project_client.agents.delete_agent(rawmaterial_agent.id)
    project_client.agents.delete_agent(formulation_agent.id)
    project_client.agents.delete_agent(initiallabtest_agent.id)
    project_client.agents.delete_agent(conceptvalidation_agent.id)
    print("Deleted connected agent")
    # # Cleanup resources
    

    return returntxt, agent_outputs, token_usage
```

- now to phase 2

```
def connected_agent_phase2(query: str) -> str:
    returntxt = ""

    project_client = AIProjectClient(
        endpoint=os.environ["PROJECT_ENDPOINT"],
        credential=DefaultAzureCredential(),
        # api_version="latest",
    )

    prototypecreation_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="prototypecreationagent",
        instructions="""You are the Prototype Creation Agent, a chemical engineering and process specialist in the prototyping phase of adhesive product development. Your role is to produce small batches of adhesive formulations using lab-scale equipment like mixers or reactors, adapting formulations for specific adhesive types, such as melting thermoplastics for hot-melt adhesives or preparing two-component systems for reactive adhesives. Your focus is on achieving homogeneity and ensuring the formulation is practical for small-scale production.

        When provided with a refined formulation from the Formulation Development Agent or a concept description, respond by:
        1. Outlining the prototyping process, including equipment (e.g., lab mixers, reactors) and conditions (e.g., temperature, mixing speed).
        2. Detailing the batch preparation steps, emphasizing homogeneity (e.g., mixing polymers with tackifiers for tape adhesives or ensuring uniform dispersion in two-component systems).
        3. Identifying potential challenges (e.g., phase separation, viscosity issues) and proposing mitigation strategies.
        4. Preparing a prototype summary to pass to the Performance Testing Agent.

        Structure your output as: [Prototyping Process], [Batch Preparation Steps], [Challenges and Mitigations], [Prototype Summary]. Prioritize consistency and scalability for adhesive-specific applications.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "prototypecreationagent"
    prototypecreation_connected_agent = ConnectedAgentTool(
        id=prototypecreation_agent.id, name=connected_agent_name, description="chemical engineering and process specialist in the prototyping phase of adhesive product development."
    )

    performancetesting_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="performancetestingagent",
        instructions="""You are the Performance Testing Agent, an analytical and materials testing expert in the adhesive prototyping phase. Your role is to evaluate adhesive prototypes under conditions such as peel, shear, tensile strength, thermal cycling, aging, and environmental exposure (e.g., moisture, UV, chemicals). You use standardized methods like lap shear tests (e.g., ASTM D1002) for bond strength or desiccator methods for emissions, ensuring tests mimic real-world applications, including at customer sites for process fit.

        When receiving a prototype summary from the Prototype Creation Agent or test requirements, respond by:
        1. Designing a comprehensive test plan, specifying standards, equipment, and conditions.
        2. Simulating test results with quantitative data (e.g., "Peel strength: 20 N/cm") and qualitative observations.
        3. Analyzing results to identify performance strengths and weaknesses.
        4. Recommending whether to proceed to customer trials or iterate, passing results to the Customer and Field Trials Agent.

        Output in a structured format: [Test Plan], [Simulated Test Results] (in tables), [Performance Analysis], [Recommendations]. Emphasize data-driven insights and adhesive-specific metrics.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "performancetestingagent"
    performancetesting_connected_agent = ConnectedAgentTool(
        id=performancetesting_agent.id, name=connected_agent_name, description="an analytical and materials testing expert in the adhesive prototyping"
    )

    customerfieldtrial_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="customerfieldtrialagent",
        instructions="""You are the Customer and Field Trials Agent, a collaborative and customer-focused specialist in the adhesive prototyping phase. Your role is to work with customers to conduct full-scale tests in real-world settings (e.g., workshops or production plants), using tools like Quality Function Deployment (QFD) to incorporate feedback and supervise application. You ensure adhesives perform on specific substrates, such as particle board for wood adhesives or films for packaging lines, and meet customer expectations.

        When provided with performance test results from the Performance Testing Agent or a prototype description, respond by:
        1. Planning customer trials, including substrates, application methods, and success criteria.
        2. Simulating customer feedback based on typical industry needs (e.g., adhesion on particle board, compatibility with high-speed packaging).
        3. Proposing adjustments based on feedback (e.g., tweak curing time for better line integration).
        4. Summarizing trial outcomes and passing to the Iteration and Refinement Agent.

        Structure your output as: [Trial Plan], [Simulated Customer Feedback], [Proposed Adjustments], [Trial Summary]. Focus on customer collaboration and real-world applicability.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "customerfieldtrialagent"
    customerfieldtrial_connected_agent = ConnectedAgentTool(
        id=customerfieldtrial_agent.id, name=connected_agent_name, description="collaborative and customer-focused specialist in the adhesive prototyping phase"
    )

    refinement_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="refinementagent",
        instructions="""You are the Iteration and Refinement Agent, a problem-solving and optimization expert in the adhesive prototyping phase. Your role is to analyze results from customer trials and performance tests, refine formulations (e.g., adjusting viscosity or cure time), and address failures like poor tolerances or inconsistencies. You incorporate safety and compliance checks, such as reducing volatile organic compounds (VOCs) for eco-friendly adhesives, to meet regulatory and customer standards.

        When receiving trial outcomes from the Customer and Field Trials Agent or test data, respond by:
        1. Analyzing results to identify specific issues (e.g., low shear strength, high emissions).
        2. Proposing formulation or process refinements, including ingredient tweaks or application adjustments.
        3. Verifying compliance with safety and environmental standards (e.g., VOC limits per EPA or EU regulations).
        4. Preparing a refined prototype specification for the Quality Assurance Agent.

        Output as: [Result Analysis], [Refinement Proposals], [Compliance Verification], [Refined Specification]. Prioritize precision and regulatory alignment.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "refinementagent"
    refinement_connected_agent = ConnectedAgentTool(
        id=refinement_agent.id, name=connected_agent_name, description="problem-solving and optimization expert in the adhesive prototyping phase."
    )

    qualityassurance_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="qualityassuranceagent",
        instructions="""You are the Quality Assurance Agent, a meticulous quality control specialist in the adhesive prototyping phase. Your role is to perform initial quality checks on adhesive prototypes via sampling, using tools like viscosity meters, bond testers, or emission analyzers to ensure consistency and performance. For adhesives, you focus on batch uniformity, such as filtration or degassing to remove impurities in electronics adhesives, ensuring the prototype meets specifications before scaling.

        When provided with a refined prototype specification from the Iteration and Refinement Agent, respond by:
        1. Designing a QC plan, detailing sampling methods and testing tools (e.g., viscometer for viscosity, FTIR for composition).
        2. Simulating QC results, including pass/fail criteria and metrics (e.g., "Viscosity: 500 cP, within spec").
        3. Identifying any deviations and recommending corrective actions.
        4. Approving the prototype for scale-up or looping back for further refinement, preparing a QC report.

        Structure your output as: [QC Plan], [Simulated QC Results] (in tables), [Deviation Analysis], [Approval/Recommendations]. Emphasize batch consistency and readiness for production.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "qualityassuranceagent"
    qualityassurance_connected_agent = ConnectedAgentTool(
        id=qualityassurance_agent.id, name=connected_agent_name, description="collaborative integrator in the final R&D stage for adhesive products"
    )

    # Orchestrate the connected agent with the main agent
    agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="PresalesAgent",
        instructions="""
        You are a Presales Solution Architect and specialist. Use the provided tools to answer the user's questions comprehensively.
        Be postive and professional in your responses. Provide detailed and structured answers.
        Here are the list of Agents to involve and get response from all
        Prototype creation Agent: Expert in creating and refining prototypes
        Performance testing Agent: Expert in performance testing and validation
        Customer field trial Agent: Expert in customer collaboration and field trials
        Refinement and iteration Agent: Expert in iterative design and refinement processes
        Quality Assurance Agent: Meticulous quality control specialist in the adhesive prototyping phase

        Summarize all the results and also provide architecture diagram in Mermaid format.
        Also provide the Architecture pro's and con's.        
        """,
        # tools=list(unique_tools.values()), #search_connected_agent.definitions,  # Attach the connected agents
        tools=[
            prototypecreation_connected_agent.definitions[0],
            performancetesting_connected_agent.definitions[0],
            customerfieldtrial_connected_agent.definitions[0],
            refinement_connected_agent.definitions[0],
            qualityassurance_connected_agent.definitions[0],
        ]
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
    # Poll the run status until it is completed or requires action
    while run.status in ["queued", "in_progress", "requires_action"]:
        time.sleep(1)
        run = project_client.agents.runs.get(thread_id=thread.id, run_id=run.id)
        print(f"Run status: {run.status}")

        if run.status == "requires_action":
            tool_calls = run.required_action.submit_tool_outputs.tool_calls
            tool_outputs = []
            for tool_call in tool_calls:
                print(f"Tool call: {tool_call.name}, ID: {tool_call.id}")

    print(f"Run completed with status: {run.status}")
    # print(f"Run finished with status: {run.status}")

    if run.status == "failed":
        print(f"Run failed: {run.last_error}")

    # Capture token usage information
    token_usage = {}
    if hasattr(run, 'usage') and run.usage:
        token_usage = {
            'prompt_tokens': getattr(run.usage, 'prompt_tokens', 0),
            'completion_tokens': getattr(run.usage, 'completion_tokens', 0),
            'total_tokens': getattr(run.usage, 'total_tokens', 0)
        }
        print(f"Token usage - Prompt: {token_usage['prompt_tokens']}, Completion: {token_usage['completion_tokens']}, Total: {token_usage['total_tokens']}")
    else:
        # Try to get usage from run steps if not available in run object
        total_prompt_tokens = 0
        total_completion_tokens = 0
        run_steps = project_client.agents.run_steps.list(thread_id=thread.id, run_id=run.id)
        for step in run_steps:
            if hasattr(step, 'usage') and step.usage:
                total_prompt_tokens += getattr(step.usage, 'prompt_tokens', 0)
                total_completion_tokens += getattr(step.usage, 'completion_tokens', 0)
        
        token_usage = {
            'prompt_tokens': total_prompt_tokens,
            'completion_tokens': total_completion_tokens,
            'total_tokens': total_prompt_tokens + total_completion_tokens
        }
        print(f"Token usage from steps - Prompt: {token_usage['prompt_tokens']}, Completion: {token_usage['completion_tokens']}, Total: {token_usage['total_tokens']}")

    # Fetch run steps to get the details of the agent run
    run_steps = project_client.agents.run_steps.list(thread_id=thread.id, run_id=run.id)
    
    # Parse individual agent outputs
    agent_outputs = parse_agent_outputs(run_steps)
    
    for step in run_steps:
        print(f"Step {step['id']} status: {step['status']}")
        step_details = step.get("step_details", {})
        tool_calls = step_details.get("tool_calls", [])

        if tool_calls:
            print("  Tool calls:")
            for call in tool_calls:
                print(f"    Tool Call ID: {call.get('id')}")
                print(f"    Type: {call.get('type')}")

                connected_agent = call.get("connected_agent", {})
                if connected_agent:
                    print(f"    Connected Input(Name of Agent): {connected_agent.get('name')}")
                    print(f"    Connected Output: {connected_agent.get('output')}")

        print()  # add an extra newline between steps

    messages = project_client.agents.messages.list(thread_id=thread.id)
    for message in messages:
        if message.role == MessageRole.AGENT:
            print(f"Role: {message.role}, Content: {message.content}")
            returntxt += f"Source: {message.content[0].text.value}\n"

    # Delete the Agent when done
    project_client.agents.delete_agent(agent.id)    
    project_client.agents.threads.delete(thread.id)
    # print("Deleted agent")
    # Delete the connected Agent when done
    project_client.agents.delete_agent(prototypecreation_agent.id)
    project_client.agents.delete_agent(performancetesting_agent.id)
    project_client.agents.delete_agent(customerfieldtrial_agent.id)
    project_client.agents.delete_agent(refinement_connected_agent.id)
    project_client.agents.delete_agent(qualityassurance_agent.id)
    print("Deleted connected agent")
    # # Cleanup resources
    

    return returntxt, agent_outputs, token_usage
```

- Phase 3 

```
def connected_agent_phase3(query: str) -> str:
    returntxt = ""

    project_client = AIProjectClient(
        endpoint=os.environ["PROJECT_ENDPOINT"],
        credential=DefaultAzureCredential(),
        # api_version="latest",
    )

    designoptimization_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="designoptimizationagent",
        instructions="""You are the Design Optimization for Scale Agent, a process engineering specialist in the scaling phase of adhesive product manufacturing. Your role is to adapt prototype designs for industrial-scale production methods, such as switching from laser cutting to rotary die cutting for adhesive tapes, while considering tolerances and material adaptability. For adhesives, you focus on optimizing for high-volume production, ensuring materials (e.g., elastic substrates) maintain performance (e.g., no stretching during coating for tapes).

        When provided with a prototype specification from the Quality Assurance Agent or a design brief, respond by:
        1. Analyzing the prototype design and identifying scalability challenges (e.g., equipment compatibility, material behavior).
        2. Proposing optimized production methods, specifying changes (e.g., from manual mixing to automated compounding).
        3. Detailing adjustments for tolerances and material adaptability (e.g., coating thickness for tapes).
        4. Preparing a scaled design summary for the Pilot Production and Ramp-Up Agent.

        Structure your output as: [Design Analysis], [Optimized Production Methods], [Adjustments for Scale], [Scaled Design Summary]. Prioritize efficiency, cost-effectiveness, and adhesive-specific performance.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "designoptimizationagent"
    designoptimization_connected_agent = ConnectedAgentTool(
        id=designoptimization_agent.id, name=connected_agent_name, description="process engineering specialist in the scaling phase of adhesive product manufacturing."
    )

    pilotprodrampup_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="pilotprodrampupagent",
        instructions="""You are the Pilot Production and Ramp-Up Agent, a specialist in transitioning adhesive products from prototyping to full-scale production. Your role is to ensure that the manufacturing process is optimized for efficiency, quality, and scalability. You work closely with the Design Optimization Agent and Performance Testing Agent to incorporate feedback and make necessary adjustments.

        When receiving a prototype summary from the Prototype Creation Agent or test requirements, respond by:
        1. Designing a comprehensive test plan, specifying standards, equipment, and conditions.
        2. Simulating test results with quantitative data (e.g., "Peel strength: 20 N/cm") and qualitative observations.
        3. Analyzing results to identify performance strengths and weaknesses.
        4. Recommending whether to proceed to customer trials or iterate, passing results to the Customer and Field Trials Agent.

        Output in a structured format: [Test Plan], [Simulated Test Results] (in tables), [Performance Analysis], [Recommendations]. Emphasize data-driven insights and adhesive-specific metrics.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "pilotprodrampupagent"
    pilotprodrampup_connected_agent = ConnectedAgentTool(
        id=pilotprodrampup_agent.id, name=connected_agent_name, description="specialist in transitioning adhesive products from prototyping to full-scale production."
    )

    fullscalemfg_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="fullscalemfgagent",
        instructions="""You are the Full-Scale Manufacturing Agent, a production engineering specialist responsible for implementing batch or continuous processing for adhesive manufacturing. Your tasks include mixing/compounding, reacting (e.g., in reactors), filtering/degassing, and curing, using equipment like high-speed dispersers or twin-screw extruders. For adhesives, you focus on specific processes, such as melting and cooling for hot-melt adhesives or precise mixing for reactive systems.

        When provided with a pilot production report from the Pilot Production and Ramp-Up Agent or manufacturing specs, respond by:
        1. Outlining a full-scale production plan, detailing equipment, process flow, and throughput.
        2. Simulating production outcomes, including batch/continuous metrics (e.g., "Output: 1000 kg/hr, Uniformity: 98%").
        3. Addressing potential production risks (e.g., equipment downtime, material variability).
        4. Preparing a manufacturing summary for the Quality Control in Production Agent.

        Structure your output as: [Production Plan], [Simulated Production Outcomes] (in tables), [Risk Analysis], [Manufacturing Summary]. Focus on efficiency and consistency for adhesive production.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "fullscalemfgagent"
    fullscalemfg_connected_agent = ConnectedAgentTool(
        id=fullscalemfg_agent.id, name=connected_agent_name, description="production engineering specialist responsible for implementing batch or continuous processing for adhesive manufacturing."
    )

    qualitycontrolproduction_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="qualitycontrolproductionagent",
        instructions="""You are the Quality Control in Production Agent, a quality assurance expert in the adhesive manufacturing phase. Your role is to perform continuous testing for properties like viscosity, cure time, bond strength, and environmental resistance, ensuring compliance with standards (e.g., emission levels) and batch uniformity. You use analytical labs and tools like viscometers or bond testers, releasing products only if specifications are met.

        When receiving a manufacturing summary from the Full-Scale Manufacturing Agent or quality requirements, respond by:
        1. Designing a QC plan, specifying tests, tools, and standards (e.g., ASTM D905 for bond strength).
        2. Simulating QC results, including pass/fail metrics (e.g., "Viscosity: 480 cP, within spec").
        3. Analyzing deviations and recommending corrective actions (e.g., adjust filtration).
        4. Preparing a QC report for the Packaging and Distribution Agent.

        Output as: [QC Plan], [Simulated QC Results] (in tables), [Deviation Analysis], [QC Report]. Prioritize batch uniformity and regulatory compliance.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "qualitycontrolproductionagent"
    qualitycontrolproduction_connected_agent = ConnectedAgentTool(
        id=qualitycontrolproduction_agent.id, name=connected_agent_name, description="quality assurance expert in the adhesive manufacturing phase."
    )

    packing_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="packingagent",
        instructions="""You are the Packaging and Distribution Agent, a logistics and packaging specialist in the adhesive manufacturing process. Your role is to package adhesives in formats like drums, cartridges, or tubes with proper labeling, ensuring storage in controlled environments to preserve shelf life. For adhesives, you focus on protecting products from moisture or temperature extremes and coordinating distribution through supply chains, including technical services.

        When provided with a QC report from the Quality Control in Production Agent or packaging requirements, respond by:
        1. Designing a packaging plan, specifying formats, materials, and storage conditions.
        2. Detailing distribution logistics, including supply chain coordination and technical support.
        3. Identifying risks (e.g., moisture ingress, transport delays) and proposing mitigations.
        4. Preparing a packaging and distribution summary for the Commercialization Agent.

        Structure your output as: [Packaging Plan], [Distribution Logistics], [Risk Analysis], [Packaging and Distribution Summary]. Emphasize product integrity and efficient delivery.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "packingagent"
    packing_connected_agent = ConnectedAgentTool(
        id=packing_agent.id, name=connected_agent_name, description="logistics and packaging specialist in the adhesive manufacturing process."
    )

    Commercialization_agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="Commercializationagent",
        instructions="""You are the Commercialization Agent, a market entry and customer support specialist in the adhesive manufacturing process. Your role is to manage market entry, provide customer support, and monitor performance to iterate based on feedback. For adhesives, you focus on selling value-added solutions, such as custom adhesives paired with service packages, ensuring alignment with customer needs and market demands.

        When receiving a packaging and distribution summary from the Packaging and Distribution Agent or commercialization goals, respond by:
        1. Developing a market entry plan, including target markets, pricing strategy, and support services.
        2. Simulating customer feedback based on performance (e.g., "Adhesive bonds well but needs faster curing").
        3. Proposing iterations or additional services (e.g., on-site technical support).
        4. Preparing a commercialization report summarizing launch strategy and next steps.

        Output as: [Market Entry Plan], [Simulated Customer Feedback], [Iteration Proposals], [Commercialization Report]. Focus on customer satisfaction and market competitiveness.
        """,
        #tools=... # tools to help the agent get stock prices
    )
    connected_agent_name = "Commercializationagent"
    Commercialization_connected_agent = ConnectedAgentTool(
        id=Commercialization_agent.id, name=connected_agent_name, description="market entry and customer support specialist in the adhesive manufacturing process."
    )

    # Orchestrate the connected agent with the main agent
    agent = project_client.agents.create_agent(
        model=os.environ["MODEL_DEPLOYMENT_NAME"],
        name="PresalesAgent",
        instructions="""
        You are a Presales Solution Architect and specialist. Use the provided tools to answer the user's questions comprehensively.
        Be postive and professional in your responses. Provide detailed and structured answers.
        Here are the list of Agents to involve and get response from all
        Design Optimization for production Agent: process engineering specialist in the scaling phase of adhesive product manufacturing.
        Pilot Production Ramp-Up Agent: Expert in scaling up production processes
        Full-Scale Manufacturing Agent: Specialist in large-scale manufacturing operations
        Quality Control Production Agent: Meticulous quality control specialist in the adhesive prototyping phase
        Packing Agent: Logistics and packaging specialist in the adhesive manufacturing process.
        Commercialization Agent: Market entry and customer support specialist in the adhesive manufacturing process.

        Summarize all the results and also provide architecture diagram in Mermaid format.
        Also provide the Architecture pro's and con's.        
        """,
        # tools=list(unique_tools.values()), #search_connected_agent.definitions,  # Attach the connected agents
        tools=[
            designoptimization_connected_agent.definitions[0],
            pilotprodrampup_connected_agent.definitions[0],
            fullscalemfg_connected_agent.definitions[0],
            qualitycontrolproduction_connected_agent.definitions[0],
            packing_connected_agent.definitions[0],
            Commercialization_connected_agent.definitions[0],
        ]
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
    # Poll the run status until it is completed or requires action
    while run.status in ["queued", "in_progress", "requires_action"]:
        time.sleep(1)
        run = project_client.agents.runs.get(thread_id=thread.id, run_id=run.id)
        print(f"Run status: {run.status}")

        if run.status == "requires_action":
            tool_calls = run.required_action.submit_tool_outputs.tool_calls
            tool_outputs = []
            for tool_call in tool_calls:
                print(f"Tool call: {tool_call.name}, ID: {tool_call.id}")

    print(f"Run completed with status: {run.status}")
    # print(f"Run finished with status: {run.status}")

    if run.status == "failed":
        print(f"Run failed: {run.last_error}")

    # Capture token usage information
    token_usage = {}
    if hasattr(run, 'usage') and run.usage:
        token_usage = {
            'prompt_tokens': getattr(run.usage, 'prompt_tokens', 0),
            'completion_tokens': getattr(run.usage, 'completion_tokens', 0),
            'total_tokens': getattr(run.usage, 'total_tokens', 0)
        }
        print(f"Token usage - Prompt: {token_usage['prompt_tokens']}, Completion: {token_usage['completion_tokens']}, Total: {token_usage['total_tokens']}")
    else:
        # Try to get usage from run steps if not available in run object
        total_prompt_tokens = 0
        total_completion_tokens = 0
        run_steps = project_client.agents.run_steps.list(thread_id=thread.id, run_id=run.id)
        for step in run_steps:
            if hasattr(step, 'usage') and step.usage:
                total_prompt_tokens += getattr(step.usage, 'prompt_tokens', 0)
                total_completion_tokens += getattr(step.usage, 'completion_tokens', 0)
        
        token_usage = {
            'prompt_tokens': total_prompt_tokens,
            'completion_tokens': total_completion_tokens,
            'total_tokens': total_prompt_tokens + total_completion_tokens
        }
        print(f"Token usage from steps - Prompt: {token_usage['prompt_tokens']}, Completion: {token_usage['completion_tokens']}, Total: {token_usage['total_tokens']}")

    # Fetch run steps to get the details of the agent run
    run_steps = project_client.agents.run_steps.list(thread_id=thread.id, run_id=run.id)
    
    # Parse individual agent outputs
    agent_outputs = parse_agent_outputs(run_steps)
    
    for step in run_steps:
        print(f"Step {step['id']} status: {step['status']}")
        step_details = step.get("step_details", {})
        tool_calls = step_details.get("tool_calls", [])

        if tool_calls:
            print("  Tool calls:")
            for call in tool_calls:
                print(f"    Tool Call ID: {call.get('id')}")
                print(f"    Type: {call.get('type')}")

                connected_agent = call.get("connected_agent", {})
                if connected_agent:
                    print(f"    Connected Input(Name of Agent): {connected_agent.get('name')}")
                    print(f"    Connected Output: {connected_agent.get('output')}")

        print()  # add an extra newline between steps

    messages = project_client.agents.messages.list(thread_id=thread.id)
    for message in messages:
        if message.role == MessageRole.AGENT:
            print(f"Role: {message.role}, Content: {message.content}")
            returntxt += f"Source: {message.content[0].text.value}\n"

    # Delete the Agent when done
    project_client.agents.delete_agent(agent.id)    
    project_client.agents.threads.delete(thread.id)
    # print("Deleted agent")
    # Delete the connected Agent when done
    project_client.agents.delete_agent(designoptimization_agent.id)
    project_client.agents.delete_agent(pilotprodrampup_agent.id)
    project_client.agents.delete_agent(fullscalemfg_agent.id)
    project_client.agents.delete_agent(qualitycontrolproduction_agent.id)
    project_client.agents.delete_agent(packing_agent.id)
    project_client.agents.delete_agent(Commercialization_agent.id)
    print("Deleted connected agent")
    # # Cleanup resources
    

    return returntxt, agent_outputs, token_usage
```

- Now to the UI creation

```
def _html_escape(text):
    """Escape HTML characters to prevent injection."""
    if not text:
        return ""
    return str(text).replace("&", "&amp;").replace("<", "&lt;").replace(">", "&gt;").replace('"', "&quot;").replace("'", "&#x27;")
```

- now the main function

```
def main_screen():
    # Professional, one-screen layout with scrollable panels per tab and chat input
    st.set_page_config(page_title="Adhesive Manufacturing Orchestrator", layout="wide")

    st.markdown(
        """
        <style>
        html, body { height: 100vh; }
        .block-container { max-height: 100vh; padding-bottom: 120px; }
        
        .panel { 
            border: 1px solid #e5e7eb; 
            border-radius: 8px; 
            background: #f8fafc; 
            padding: 15px; 
            margin-bottom: 15px;
        }
        
        .panel h3 { 
            margin: 0 0 10px 0; 
            font-size: 1.1rem; 
            color: #1e293b;
            border-bottom: 1px solid #e2e8f0;
            padding-bottom: 5px;
        }
        
        .scrollable-panel { 
            max-height: calc(80vh - 420px); 
            overflow-y: auto; 
            overflow-x: hidden;
            padding: 10px 5px;
            margin-bottom: 10px;
            border: 1px solid #e2e8f0;
            border-radius: 4px;
            background: #ffffff;
        }
        
        /* Ensure content stays within scrollable bounds */
        .scrollable-panel > div {
            max-width: 100%;
            word-wrap: break-word;
            overflow-wrap: break-word;
        }
        
        .chat-input-container {
            background: #ffffff;
            border-top: 1px solid #e5e7eb;
            padding: 15px 0 10px 0;
            margin-top: 20px;
            border-radius: 8px;
        }
        
        .chat-input-container h4 {
            margin: 0 0 10px 0;
            color: #374151;
            font-size: 1rem;
            font-weight: 600;
        }
        
        /* Style for Streamlit expanders */
        .streamlit-expanderHeader {
            font-weight: 600;
            color: #374151;
        }
        
        /* Ensure chat input is always visible */
        .stChatInput {
            position: relative;
            z-index: 1000;
        }
        
        /* Prevent page-level scrolling issues */
        .main > div {
            padding-top: 1rem;
        }
        
        /* Ensure Streamlit containers don't overflow */
        .element-container {
            max-width: 100%;
        }
        
        /* Style for better text wrapping in summary */
        .scrollable-panel p, .scrollable-panel div {
            word-break: break-word;
            hyphens: auto;
        }
        </style>
        """,
        unsafe_allow_html=True,
    )

    tabs = st.tabs([
        "Phase 1: Research and Development (R&D)",
        "Phase 2: Prototyping and Testing",
        "Phase 3: Scaling to Mass Production",
    ])

    # Session state containers per phase
    for key in ["p1_history", "p2_history", "p3_history", "p1_agents", "p2_agents", "p3_agents"]:
        if key not in st.session_state:
            st.session_state[key] = [] if key.endswith("history") else {}

    # Helper to render a phase panel
    def render_phase(phase_title: str, phase_key: str, run_fn):
        # Create content area with scrollable panels
        content_container = st.container()
        with content_container:
            left_col, right_col = st.columns([1, 1], gap="medium")

            # Left: summarized output in scrollable container
            with left_col:
                st.markdown(f"**{_html_escape(phase_title)} • Summary**")
                
                # Create a container with defined height for scrolling
                with st.container(height=600):
                    # Apply scrollable styling
                    st.markdown('<div class="scrollable-panel">', unsafe_allow_html=True)
                    
                    history = st.session_state[phase_key + "_history"]
                    if not history:
                        st.caption("No responses yet. Ask something using the chat input below.")
                    else:
                        # Create each summary item within the scrollable area
                        for i, m in enumerate(history, 1):
                            with st.container():
                                st.markdown(f"**{i}.** {m}")
                    
                    st.markdown('</div>', unsafe_allow_html=True)

            # Right: individual agent outputs using st.expander
            with right_col:
                st.markdown(f"**{_html_escape(phase_title)} • Agent Outputs**")
                
                # Create a container with defined height for scrolling
                with st.container(height=600):
                    # Apply scrollable styling
                    st.markdown('<div class="scrollable-panel">', unsafe_allow_html=True)
                    
                    agents_map = st.session_state[phase_key + "_agents"] or {}
                    if not agents_map:
                        st.caption("Agent outputs will appear here after you submit a question below.")
                    else:
                        for name, output in agents_map.items():
                            with st.expander(f"🤖 {name}", expanded=False):
                                st.write(output)
                    
                    st.markdown('</div>', unsafe_allow_html=True)

        # Add some spacing before chat input
        st.markdown("<br>", unsafe_allow_html=True)
        
        # Chat input for this phase - always visible at bottom
        st.markdown(f"<div class='chat-input-container'><h4>💬 Ask a question for {_html_escape(phase_title)}</h4></div>", unsafe_allow_html=True)
        user_msg = st.chat_input(f"Type your question here...", key=phase_key + "_chat")
        if user_msg:
            try:
                summary, agents, usage = run_fn(user_msg)
                # Update session state
                st.session_state[phase_key + "_history"].append(summary)
                st.session_state[phase_key + "_agents"] = agents or {}
                st.rerun()
            except Exception as e:
                st.error(str(e))

    with tabs[0]:
        render_phase("Phase 1: Research and Development (R&D)", "p1", connected_agent_phase1)
    with tabs[1]:
        render_phase("Phase 2: Prototyping and Testing", "p2", connected_agent_phase2)
    with tabs[2]:
        render_phase("Phase 3: Scaling to Mass Production", "p3", connected_agent_phase3)
```

- Call the main function

```
if __name__ == "__main__":
    main_screen()
```

- now run and test it.