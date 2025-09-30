# Grok 4 Fast Reasoning in Azure AI Foundry

## Introduction

- Grok 4 is a powerful language model designed for fast reasoning tasks.
- This guide provides instructions on how to integrate and utilize Grok 4 within the Azure AI Foundry environment.
- using Open AI chat completions
- Create a UI for Grok 4 in Azure AI Foundry
- Working on streamlit app

## Prerequisites

- Azure Subscription
- Azure AI Foundry Project
- Deploy - grok-4-fast-reasoning
- i am using East us2 region
- Python 3.12+

## Code

- Here is the code to use grok-4-fast-reasoning in Azure AI Foundry

```
import datetime
import os, time, json
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from openai import AzureOpenAI
from azure.ai.agents.models import (
    ListSortOrder,
    McpTool,
    RequiredMcpToolCall,
    RunStepActivityDetails,
    SubmitToolApprovalAction,
    ToolApproval,
    FunctionTool,
    MessageRole,
    ConnectedAgentTool,
    FilePurpose,
)
from azure.ai.agents.models import AzureAISearchTool, AzureAISearchQueryType, MessageRole, ListSortOrder, ToolDefinition, FilePurpose, FileSearchTool
from azure.ai.agents.models import CodeInterpreterTool, FunctionTool, ToolSet
import requests
import streamlit as st
from datetime import datetime, timedelta
import pandas as pd
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

endpoint = os.environ["PROJECT_ENDPOINT"] # Sample : https://<account_name>.services.ai.azure.com/api/projects/<project_name>
model_endpoint = os.environ["MODEL_ENDPOINT"] # Sample : https://<account_name>.services.ai.azure.com
model_api_key= os.environ["MODEL_API_KEY"]
model_deployment_name = os.environ["MODEL_DEPLOYMENT_NAME"] # Sample : gpt-4o-mini

# Get MCP server configuration from environment variables
mcp_server_url = os.environ.get("MCP_SERVER_URL", "https://learn.microsoft.com/api/mcp")
mcp_server_label = os.environ.get("MCP_SERVER_LABEL", "MicrosoftLearn")

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
# connection_string = project_client.telemetry.get_application_insights_connection_string()
connection_string = os.getenv("APPLICATION_INSIGHTS_CONNECTION_STRING")

if not connection_string:
    print("Application Insights is not enabled. Enable by going to Tracing in your Azure AI Foundry project.")
    exit()

configure_azure_monitor(connection_string=connection_string) #enable telemetry collection

from opentelemetry import trace
tracer = trace.get_tracer(__name__)

def grok_query(messages: list):
    """Send chat history to model. Returns (reply_text, usage_dict, latency_seconds)."""
    deployment_name = "grok-4-fast-reasoning"
    start = time.perf_counter()
    try:
        completion = client.chat.completions.create(
            model=deployment_name,
            messages=messages,
        )
        latency = time.perf_counter() - start
        # Extract usage safely (Azure/OpenAI style compatibility)
        usage_obj = getattr(completion, 'usage', None) or getattr(completion, 'model_usage', None) or {}
        if isinstance(usage_obj, dict):
            prompt_tokens = usage_obj.get('prompt_tokens') or usage_obj.get('input_tokens') or 0
            completion_tokens = usage_obj.get('completion_tokens') or usage_obj.get('output_tokens') or 0
            total_tokens = usage_obj.get('total_tokens') or (prompt_tokens + completion_tokens)
        else:
            # Fallback attributes
            prompt_tokens = getattr(usage_obj, 'prompt_tokens', 0)
            completion_tokens = getattr(usage_obj, 'completion_tokens', 0)
            total_tokens = getattr(usage_obj, 'total_tokens', prompt_tokens + completion_tokens)
        usage = {
            'prompt_tokens': prompt_tokens,
            'completion_tokens': completion_tokens,
            'total_tokens': total_tokens
        }
        reply = completion.choices[0].message.content
        return reply, usage, latency
    except Exception as e:
        latency = time.perf_counter() - start
        return f"[ERROR] {e}", {'prompt_tokens': 0, 'completion_tokens': 0, 'total_tokens': 0}, latency

def init_session_state():
    if 'chat_history' not in st.session_state:
        st.session_state.chat_history = []  # list of {role, content}
    if 'system_prompt' not in st.session_state:
        st.session_state.system_prompt = "You are a helpful assistant. Keep answers concise unless detail is requested."
    if 'metrics' not in st.session_state:
        st.session_state.metrics = {
            'total_prompt_tokens': 0,
            'total_completion_tokens': 0,
            'total_tokens': 0,
            'total_requests': 0,
            'last_latency': None,
            'last_usage': None,
        }

def safe_rerun():
    """Call the appropriate rerun method depending on Streamlit version."""
    if hasattr(st, 'rerun'):
        st.rerun()
    else:
        # Fallback for older versions
        getattr(st, 'experimental_rerun')()

def render_chat_container(height_px: int = 500):
    """Render chat history inside a scrollable container of fixed height (default 500px)."""
    chat_css = f"""
    <style>
    .chat-box {{max-height:{height_px}px; overflow-y:auto; padding:0.75rem; border:1px solid #d9d9d9; border-radius:10px; background:#fafafa;}}
    .chat-msg {{margin-bottom:0.9rem; line-height:1.35;}}
    .chat-role-user strong {{color:#1d4ed8;}}
    .chat-role-assistant strong {{color:#047857;}}
    .chat-role-system strong {{color:#6d28d9;}}
    .chat-tokens {{font-size:0.7rem; opacity:0.6;}}
    </style>
    """
    st.markdown(chat_css, unsafe_allow_html=True)
    with st.container(height=height_px):
        html_parts = ["<div class='chat-box'>"]
        for i, msg in enumerate(st.session_state.chat_history):
            role = msg.get('role','assistant')
            safe_content = msg.get('content','')
            html_parts.append(
                f"<div class='chat-msg chat-role-{role}'><strong>{role.capitalize()}:</strong> {safe_content}</div>"
            )
        html_parts.append("</div>")
        st.markdown("\n".join(html_parts), unsafe_allow_html=True)

def add_user_message(user_text: str):
    st.session_state.chat_history.append({'role': 'user', 'content': user_text})

def add_assistant_message(assistant_text: str):
    st.session_state.chat_history.append({'role': 'assistant', 'content': assistant_text})

def build_model_messages():
    messages = []
    if st.session_state.system_prompt:
        messages.append({"role": "system", "content": st.session_state.system_prompt})
    messages.extend(st.session_state.chat_history)
    return messages

def main():
    st.set_page_config(page_title="Grok Chat", page_icon="💬", layout="centered")
    st.title("Grok Reasoning Chat")
    init_session_state()

    with st.sidebar:
        st.subheader("Settings")
        st.text_area("System Prompt", key="system_prompt", height=120,
                     help="Adjust the system behavior / persona.")
        if st.button("Clear Chat", type="secondary"):
            st.session_state.chat_history = []
            safe_rerun()

    st.caption("Enter a message below. Conversation history is sent on each turn.")

    render_chat_container()

    # Metrics display (if available)
    if 'metrics' in st.session_state and st.session_state.metrics.get('total_requests', 0) > 0:
        m = st.session_state.metrics
        with st.expander("Session Metrics", expanded=False):
            col_a, col_b, col_c, col_d = st.columns(4)
            col_a.metric("Requests", m['total_requests'])
            col_b.metric("Total Tokens", m['total_tokens'])
            col_c.metric("Prompt Tokens", m['total_prompt_tokens'])
            col_d.metric("Completion Tokens", m['total_completion_tokens'])
            if m.get('last_usage') is not None:
                st.caption(f"Last response latency: {m['last_latency']:.2f}s | Last prompt tokens: {m['last_usage']['prompt_tokens']} | Last completion tokens: {m['last_usage']['completion_tokens']} | Last total: {m['last_usage']['total_tokens']}")

    # Input form
    with st.form(key="chat_input_form", clear_on_submit=True):
        user_input = st.text_input("Your Message", placeholder="Ask me anything...", label_visibility="collapsed")
        cols = st.columns([1,1,6])
        with cols[0]:
            submit = st.form_submit_button("Send", use_container_width=True)
        with cols[1]:
            regen = st.form_submit_button("Regenerate", use_container_width=True, help="Re-send last user message")

    # Handle Send
    if submit and user_input.strip():
        add_user_message(user_input.strip())
        with st.spinner("Thinking...", show_time=True):
            messages = build_model_messages()
            reply, usage, latency = grok_query(messages)
        add_assistant_message(reply)
        # Update metrics
        m = st.session_state.metrics
        m['total_prompt_tokens'] += usage['prompt_tokens']
        m['total_completion_tokens'] += usage['completion_tokens']
        m['total_tokens'] += usage['total_tokens']
        m['total_requests'] += 1
        m['last_latency'] = latency
        m['last_usage'] = usage
        # Only rerun after processing a send to clear form and refresh display
        safe_rerun()

    # Handle Regenerate (re-use last user message)
    if regen:
        # Find last user message
        last_user = None
        for msg in reversed(st.session_state.chat_history):
            if msg['role'] == 'user':
                last_user = msg['content']
                break
        if last_user:
            # Remove trailing assistant message if any
            if st.session_state.chat_history and st.session_state.chat_history[-1]['role'] == 'assistant':
                st.session_state.chat_history.pop()
            with st.spinner("Revisiting..."):
                messages = build_model_messages()
                reply, usage, latency = grok_query(messages)
            add_assistant_message(reply)
            m = st.session_state.metrics
            m['total_prompt_tokens'] += usage['prompt_tokens']
            m['total_completion_tokens'] += usage['completion_tokens']
            m['total_tokens'] += usage['total_tokens']
            m['total_requests'] += 1
            m['last_latency'] = latency
            m['last_usage'] = usage
            safe_rerun()

    # (Removed duplicate render to avoid showing messages twice in same run)

if __name__ == "__main__":
    main()
```

- Code will also show the token usage and latency for each request
- Save this code in a file named `grok_chat.py`
- run `streamlit run grok_chat.py` to start the app
- Open the app in your browser at `http://localhost:8501`
- You can now interact with Grok 4 for fast reasoning tasks!

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/stgrok-1.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/stgrok-2.jpg 'RagChat')

## conclusion

- as of now grok-4-fast-reasoning is only available in East US2 with chat completions API
- Next will be to wait for Azure AI agents ability to support grok-4-fast-reasoning
- for learning purpose only
- Provide a streamlit app to interact with grok-4-fast-reasoning