# AGF API documentation Chat Bot

## Introduction

- Using Azure AI Foundry agents to create a chat bot.
- Answer AGF API documentation questions.
- Only answers questions related to AGF API documentation.
- Provide the schema and parameters of the API.
- Easy tool to understand AGF API.
- Pick the right API to use for your use case.

## Pre-requisites

- Azure AI Foundry account.
- AGF Account with API access.
- Using Azure AI Foundry agents.
- I am using File agent to read the JSON postman file which has API specifications.
- Goal is to create a chat bot that can answer questions about AGF API documentation.

## Requirements and environment setup

- Here are the contents of the `requirements.txt` file:

```
azure-ai-projects
azure-ai-agents
streamlit
openai
uv
python-dotenv
opentelemetry-sdk
```

- Here is what need for the environment setup:
- Create a `.env` file with the following content:

```
AZURE_OPENAI_ENDPOINT=<your-azure-openai-endpoint>
AZURE_OPENAI_API_KEY=<your-azure-openai-api-key>
PROJECT_ENDPOINT=<your-project-endpoint>
AZURE_OPENAI_DEPLOYMENT=<your-azure-openai-deployment>
```

- Replace `<your-azure-openai-endpoint>`, `<your-azure-openai-api-key>`, `<your-project-endpoint>`, and `<your-azure-openai-deployment>` with your actual Azure OpenAI endpoint, API key, project endpoint, and deployment name.

## Code to create AGF API documentation chat bot

- import libraries

```
from openai import AzureOpenAI
import streamlit as st
import fitz  # PyMuPDF  
import json
import pandas as pd
# import asyncio
import os
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.ai.agents.models import CodeInterpreterTool
from azure.ai.evaluation import AIAgentConverter
from azure.ai.agents.models import FilePurpose, MessageRole, FileSearchTool
import re
import streamlit_mermaid as stmd

# from pdf2image import convert_from_path  # Replaced with PyMuPDF
from PIL import Image
import io

from dotenv import load_dotenv
load_dotenv()

from datetime import datetime
```

- Set up Azure AI Foundry client

```
client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"), 
  api_key=os.getenv("AZURE_OPENAI_API_KEY"),  
  api_version="2024-10-21",
)

# model_name = os.getenv("AZURE_OPENAI_DEPLOYMENT_MODELROUTER")
model_name = os.getenv("AZURE_OPENAI_DEPLOYMENT")

# Azure AI Projects configuration
os.environ["AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED"] = "true" 
project_endpoint = os.environ["PROJECT_ENDPOINT"]

# Create the project client (Foundry project and credentials)
project_client = AIProjectClient(
    endpoint=project_endpoint,
    credential=DefaultAzureCredential(),
)
```

- Define the agent to load JSON and get the answers
- Using Azure AI Foundry file agent to read the JSON file containing AGF API documentation.

```
client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"), 
  api_key=os.getenv("AZURE_OPENAI_API_KEY"),  
  api_version="2024-10-21",
)

# model_name = os.getenv("AZURE_OPENAI_DEPLOYMENT_MODELROUTER")
model_name = os.getenv("AZURE_OPENAI_DEPLOYMENT")

# Azure AI Projects configuration
os.environ["AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED"] = "true" 
project_endpoint = os.environ["PROJECT_ENDPOINT"]

# Create the project client (Foundry project and credentials)
project_client = AIProjectClient(
    endpoint=project_endpoint,
    credential=DefaultAzureCredential(),
)
```

- Now the Main streamlit app for UI and interaction
- Also manage conversation history and display the chat messages

```
def agfdocsmain():
    """
    Professional business interface for API documentation assistant
    Features chat history, scrollable containers, and fits everything on one page
    """
    
    # Configure page layout
    st.set_page_config(
        page_title="API Documentation Assistant",
        page_icon="🤖",
        layout="wide",
        initial_sidebar_state="expanded"
    )
    
    # Initialize session state for chat history
    if 'chat_history' not in st.session_state:
        st.session_state.chat_history = []
    if 'query_count' not in st.session_state:
        st.session_state.query_count = 0
    
    # Professional CSS styling
    st.markdown("""
    <style>
    .main-header {
        background: linear-gradient(90deg, #1e40af, #3b82f6, #60a5fa);
        padding: 1.5rem;
        border-radius: 12px;
        margin-bottom: 1.5rem;
        text-align: center;
        color: white;
        font-size: 2.2rem;
        font-weight: bold;
        text-shadow: 2px 2px 4px rgba(0,0,0,0.3);
        box-shadow: 0 4px 20px rgba(59, 130, 246, 0.3);
    }
    
    .business-container {
        background: white;
        border-radius: 10px;
        padding: 1.5rem;
        box-shadow: 0 2px 15px rgba(0,0,0,0.1);
        border: 1px solid #e5e7eb;
        margin-bottom: 1rem;
    }
    
    .chat-message {
        padding: 1rem;
        margin: 0.8rem 0;
        border-radius: 10px;
        border-left: 4px solid #3b82f6;
        background: #f8fafc;
        box-shadow: 0 1px 3px rgba(0,0,0,0.1);
    }
    
    .user-message {
        background: linear-gradient(135deg, #1e40af 0%, #3b82f6 100%);
        color: white;
        border-left: 4px solid #1d4ed8;
        margin-left: 20%;
    }
    
    .assistant-message {
        background: linear-gradient(135deg, #f8fafc 0%, #e2e8f0 100%);
        color: #1e293b;
        border-left: 4px solid #10b981;
    }
    
    .citation-footer {
        background: #f1f5f9;
        border-top: 1px solid #d1d5db;
        padding: 0.8rem;
        margin-top: 1rem;
        border-radius: 0 0 8px 8px;
        font-size: 0.85rem;
        color: #64748b;
    }
    
    .citation-badge {
        display: inline-block;
        background: #3b82f6;
        color: white;
        padding: 0.2rem 0.6rem;
        border-radius: 12px;
        font-size: 0.75rem;
        font-weight: 500;
        margin: 0.2rem;
    }
    
    .api-spec-section {
        background: #f8fafc;
        border-left: 4px solid #10b981;
        padding: 1rem;
        margin: 0.8rem 0;
        border-radius: 0 8px 8px 0;
    }
    
    .spec-header {
        color: #059669;
        font-weight: 600;
        font-size: 1rem;
        margin-bottom: 0.5rem;
        display: flex;
        align-items: center;
        gap: 0.5rem;
    }
    
    .endpoint-badge {
        background: #1e40af;
        color: white;
        padding: 0.2rem 0.8rem;
        border-radius: 16px;
        font-size: 0.8rem;
        font-weight: 500;
        font-family: monospace;
    }
    
    .method-badge {
        padding: 0.2rem 0.6rem;
        border-radius: 4px;
        font-size: 0.75rem;
        font-weight: 600;
        font-family: monospace;
    }
    
    .method-get { background: #10b981; color: white; }
    .method-post { background: #f59e0b; color: white; }
    .method-put { background: #3b82f6; color: white; }
    .method-delete { background: #ef4444; color: white; }
    .method-patch { background: #8b5cf6; color: white; }
    
    .code-block {
        background: #1e293b;
        color: #e2e8f0;
        padding: 1rem;
        border-radius: 6px;
        font-family: 'Courier New', monospace;
        font-size: 0.85rem;
        overflow-x: auto;
        margin: 0.5rem 0;
    }
    
    .parameter-table {
        background: white;
        border: 1px solid #e5e7eb;
        border-radius: 6px;
        margin: 0.5rem 0;
    }
    
    .auth-info {
        background: #fef3c7;
        border: 1px solid #f59e0b;
        border-radius: 6px;
        padding: 0.8rem;
        margin: 0.5rem 0;
        color: #92400e;
    }
    
    .message-header {
        font-weight: bold;
        font-size: 0.9rem;
        margin-bottom: 0.5rem;
        opacity: 0.8;
    }
    
    .message-content {
        line-height: 1.6;
        font-size: 0.95rem;
    }
    
    .stats-card {
        background: linear-gradient(135deg, #f1f5f9 0%, #e2e8f0 100%);
        padding: 1rem;
        border-radius: 8px;
        text-align: center;
        box-shadow: 0 1px 3px rgba(0,0,0,0.1);
        border: 1px solid #cbd5e1;
    }
    
    .stats-number {
        font-size: 1.8rem;
        font-weight: bold;
        color: #1e40af;
        margin-bottom: 0.2rem;
    }
    
    .stats-label {
        font-size: 0.85rem;
        color: #64748b;
        font-weight: 500;
    }
    
    .input-container {
        background: white;
        padding: 1.5rem;
        border-radius: 10px;
        box-shadow: 0 2px 10px rgba(0,0,0,0.08);
        border: 1px solid #e5e7eb;
        margin-top: 1rem;
    }
    
    .stTextInput > div > div > input {
        border-radius: 8px;
        border: 2px solid #d1d5db;
        padding: 0.8rem 1rem;
        font-size: 1rem;
        transition: all 0.3s ease;
    }
    
    .stTextInput > div > div > input:focus {
        border-color: #3b82f6;
        box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
    }
    
    .stButton > button {
        background: linear-gradient(45deg, #3b82f6, #1d4ed8);
        color: white;
        border: none;
        border-radius: 8px;
        padding: 0.8rem 2rem;
        font-weight: 600;
        font-size: 1rem;
        transition: all 0.3s ease;
        box-shadow: 0 2px 8px rgba(59, 130, 246, 0.3);
    }
    
    .stButton > button:hover {
        transform: translateY(-2px);
        box-shadow: 0 4px 15px rgba(59, 130, 246, 0.4);
    }
    
    .welcome-message {
        background: linear-gradient(135deg, #eff6ff 0%, #dbeafe 100%);
        padding: 1.5rem;
        border-radius: 10px;
        border-left: 4px solid #3b82f6;
        text-align: center;
        margin: 1rem 0;
        color: #1e40af;
        font-weight: 500;
    }
    
    .empty-state {
        text-align: center;
        padding: 2rem;
        color: #64748b;
        font-style: italic;
    }
    
    .section-title {
        color: #1e293b;
        font-size: 1.1rem;
        font-weight: 600;
        margin-bottom: 1rem;
        padding-bottom: 0.5rem;
        border-bottom: 2px solid #e5e7eb;
    }
    </style>
    """, unsafe_allow_html=True)
    
    # Main header
    st.markdown('<div class="main-header">🤖 Professional API Documentation Assistant</div>', unsafe_allow_html=True)
    
    # Create main layout with sidebar
    with st.sidebar:
        st.markdown('<div class="section-title">📊 Dashboard</div>', unsafe_allow_html=True)
        
        # Statistics cards
        col1, col2 = st.columns(2)
        with col1:
            st.markdown(f"""
            <div class="stats-card">
                <div class="stats-number">{len(st.session_state.chat_history)}</div>
                <div class="stats-label">Total Messages</div>
            </div>
            """, unsafe_allow_html=True)
        
        with col2:
            st.markdown(f"""
            <div class="stats-card">
                <div class="stats-number">{st.session_state.query_count}</div>
                <div class="stats-label">API Queries</div>
            </div>
            """, unsafe_allow_html=True)
        
        st.markdown("---")
        
        # Controls
        st.markdown('<div class="section-title">🎛️ Controls</div>', unsafe_allow_html=True)
        
        if st.button("🗑️ Clear Chat History", use_container_width=True):
            st.session_state.chat_history = []
            st.session_state.query_count = 0
            st.rerun()
        
        if st.button("📋 Export Chat", use_container_width=True):
            if st.session_state.chat_history:
                chat_export = "\n".join([f"{msg['role']}: {msg['content']}" for msg in st.session_state.chat_history])
                st.download_button(
                    label="💾 Download Chat",
                    data=chat_export,
                    file_name=f"api_chat_{datetime.now().strftime('%Y%m%d_%H%M%S')}.txt",
                    mime="text/plain"
                )
        
        st.markdown("---")
        
        # Help section
        st.markdown('<div class="section-title">💡 Quick Start</div>', unsafe_allow_html=True)
        
        # File status check
        st.markdown('<div class="section-title">📁 Source Document</div>', unsafe_allow_html=True)
        file_path = "./agfdocs/HUB_FUNCTIONS_COLLECTIONS.postman_collection.json"
        file_name = os.path.basename(file_path)
        
        if os.path.exists(file_path):
            try:
                with open(file_path, 'r', encoding='utf-8') as f:
                    collection = json.load(f)
                file_size = os.path.getsize(file_path)
                item_count = len(collection.get('item', []))
                
                st.markdown(f"""
                <div class="stats-card">
                    <div class="stats-number">✅</div>
                    <div class="stats-label">Collection Loaded</div>
                </div>
                """, unsafe_allow_html=True)
                
                st.info(f"""
                **📋 File:** `{file_name}`  
                **📊 APIs:** {item_count} endpoints  
                **💾 Size:** {file_size:,} bytes  
                **🔗 Collection:** {collection.get('info', {}).get('name', 'Unknown')}
                """)
                
            except UnicodeDecodeError:
                st.error("❌ File encoding issue - Please save as UTF-8")
            except json.JSONDecodeError:
                st.error("❌ Invalid JSON file")
            except Exception as e:
                st.error(f"❌ File error: {str(e)}")
        else:
            st.error(f"❌ File not found:\n`{file_name}`")
        
        st.markdown("""
        **Sample Questions:**
        • "Show me the exact request structure for authentication APIs"
        • "What are the actual headers and parameters for search endpoints?"
        • "Extract the exact request body format for user creation"
        • "What are the actual response examples in the collection?"
        • "Show me the exact variable names and endpoints for file operations"
        
        **Features:**
        • 🤖 AI-powered JSON extraction
        • 📚 Exact file content only
        • 🔍 Precise technical specifications
        • 📊 Actual parameters from JSON
        • 🔐 Real authentication configurations
        • 💾 Exact request/response formats
        """)
    
    # Main content area - split into two columns
    col1, col2 = st.columns([3, 2])
    
    with col1:
        # Chat history section
        st.markdown('<div class="section-title">💬 Chat History</div>', unsafe_allow_html=True)
        
        # Scrollable chat container
        with st.container(height=400):
            if st.session_state.chat_history:
                for i, message in enumerate(st.session_state.chat_history):
                    timestamp = datetime.fromisoformat(message['timestamp']).strftime("%H:%M:%S")
                    
                    if message['role'] == 'user':
                        st.markdown(f"""
                        <div class="chat-message user-message">
                            <div class="message-header">👤 You • {timestamp}</div>
                            <div class="message-content">{message['content']}</div>
                        </div>
                        """, unsafe_allow_html=True)
                    else:
                        # Format assistant message with better citation handling and API spec formatting
                        content = message['content']
                        
                        # Extract citation footer if present
                        if "---\n📁 **Source:**" in content:
                            main_content, citation_info = content.split("---\n📁 **Source:**", 1)
                            citation_display = "📁 **Source:** " + citation_info
                        else:
                            main_content = content
                            citation_display = ""
                        
                        # Enhanced content processing for API specifications
                        # Convert markdown-style headers to styled divs
                        main_content = main_content.replace("## 📋", '<div class="spec-header">📋')
                        main_content = main_content.replace("## 🔧", '<div class="spec-header">🔧')
                        main_content = main_content.replace("## 🔐", '<div class="spec-header">🔐')
                        main_content = main_content.replace("## 📥", '<div class="spec-header">📥')
                        main_content = main_content.replace("## 📤", '<div class="spec-header">📤')
                        main_content = main_content.replace("## 💡", '<div class="spec-header">💡')
                        main_content = main_content.replace("## ⚙️", '<div class="spec-header">⚙️')
                        
                        # Close the header divs properly
                        main_content = main_content.replace('\n-', '</div>\n-')
                        main_content = main_content.replace('\n*', '</div>\n*')
                        
                        st.markdown(f"""
                        <div class="chat-message assistant-message">
                            <div class="message-header">🤖 API Assistant • {timestamp}</div>
                            <div class="message-content">{main_content}</div>
                            {f'<div class="citation-footer">{citation_display}</div>' if citation_display else ''}
                        </div>
                        """, unsafe_allow_html=True)
            else:
                st.markdown("""
                <div class="welcome-message">
                    <h3>👋 Welcome to the API Documentation Assistant!</h3>
                    <p>Start by asking about APIs, endpoints, or integration guidance.</p>
                    <p><strong>Example:</strong> "What are the exact specifications for authentication?"</p>
                </div>
                """, unsafe_allow_html=True)
    
    with col2:
        # Query input and information section
        st.markdown('<div class="section-title">🔍 Ask About APIs</div>', unsafe_allow_html=True)
        
        # Input container
        with st.container():
            st.markdown('<div class="input-container">', unsafe_allow_html=True)
            
            # Query input
            query = st.text_input(
                "Enter your API question:",
                value="What are the exact specifications for Search integration?",
                placeholder="e.g., Show me the exact request structure for authentication",
                help="Ask about exact API specifications, parameters, headers, and configurations from the JSON file"
            )
            
            # Submit button
            submit_col1, submit_col2 = st.columns([3, 1])
            with submit_col1:
                submit_button = st.button("🚀 Get Exact API Specifications", use_container_width=True, type="primary")
            with submit_col2:
                if st.button("📝", help="Add to favorites", use_container_width=True):
                    st.info("Feature coming soon!")
            
            st.markdown('</div>', unsafe_allow_html=True)
        
        # Quick suggestions
        st.markdown("**💡 Quick Suggestions:**")
        quick_suggestions = [
            "Exact authentication structure",
            "Request/Response JSON formats",
            "Actual headers and parameters",
            "Real endpoint specifications",
            "Collection variables and config"
        ]
        
        cols = st.columns(2)
        for i, suggestion in enumerate(quick_suggestions):
            with cols[i % 2]:
                if st.button(f"💭 {suggestion}", key=f"suggestion_{i}", use_container_width=True):
                    # Auto-fill the query
                    st.session_state.suggested_query = f"Show me the exact {suggestion.lower()} from the JSON collection"
                    st.rerun()
        
        # Check for suggested query
        if hasattr(st.session_state, 'suggested_query'):
            query = st.session_state.suggested_query
            delattr(st.session_state, 'suggested_query')
            submit_button = True
    
    # Process the query
    if submit_button and query:
        if query.strip():
            # Add user message to history
            user_message = {
                'role': 'user',
                'content': query,
                'timestamp': datetime.now().isoformat()
            }
            st.session_state.chat_history.append(user_message)
            
            # Show processing indicator
            with st.spinner("🔍 Extracting exact specifications from JSON..."):
                try:
                    # Get API recommendations
                    response = get_agf_docs(query)
                    
                    # Add assistant response to history
                    assistant_message = {
                        'role': 'assistant',
                        'content': response,
                        'timestamp': datetime.now().isoformat()
                    }
                    st.session_state.chat_history.append(assistant_message)
                    st.session_state.query_count += 1
                    
                    # Success message
                    st.success("✅ Exact API specifications extracted successfully!")
                    
                except Exception as e:
                    error_message = f"❌ Error extracting API specifications: {str(e)}"
                    assistant_message = {
                        'role': 'assistant',
                        'content': error_message,
                        'timestamp': datetime.now().isoformat()
                    }
                    st.session_state.chat_history.append(assistant_message)
                    st.error("Failed to extract API specifications. Please check your configuration.")
            
            # Refresh the page to show new messages
            st.rerun()
        else:
            st.warning("⚠️ Please enter a valid question about API specifications")
    
    # Footer with additional information
    with st.expander("ℹ️ About This Assistant", expanded=False):
        st.markdown("""
        **Professional API Documentation Assistant**
        
        This assistant extracts exact technical specifications from your Postman collection by:
        - 🔍 **Exact JSON Extraction**: Reading actual field names, values, and structures
        - 🤖 **AI-Powered Analysis**: Using Azure AI to parse and understand JSON structure
        - 📚 **File-Only Responses**: Only using data from your uploaded Postman collection
        - 💼 **Business-Ready**: Professional interface for enterprise API documentation
        
        **Powered by:**
        - Azure OpenAI for intelligent JSON analysis
        - Azure AI Projects for document processing
        - Streamlit for professional web interface
        """)
```

- Now create the main function to run the app

```
if __name__ == "__main__":
    agfdocsmain()
```

- Now run the app using streamlit

```
streamlit run app.py
```

- Type your questions about API specifications
- See the responses with exact API specifications
- Make sure output matches the content in JSON file