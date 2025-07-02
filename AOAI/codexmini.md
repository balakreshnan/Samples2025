# Using Codex Mini to do code migration

## Introduction

- Codex Mini is a tool for processing code
- Testing Cobol
- Migrating legacy code to modern languages
- Code refactoring and optimization

## Prerequisites

- Azure Subscription
- Azure AI Foundry in East US 2 region
- Deploy Codex Mini in Azure AI Foundry
- Python 3.12 or later
- Visual Studio Code or any other code editor
- Configure Environment Variables:

```
# Azure AI Foundry Project Configuration
# Replace <placeholders> with your actual values

# Core Azure AI Foundry Configuration
PROJECT_ENDPOINT=https://<account_name>.services.ai.azure.com/api/projects/<project_name>
MODEL_ENDPOINT=https://<account_name>.services.ai.azure.com
MODEL_API_KEY=<your_model_api_key>
MODEL_DEPLOYMENT_NAME=gpt-4o-mini

# Alternative project configuration (use either this or the individual settings above)
AZURE_AI_PROJECT=https://<account_name>.services.ai.azure.com/api/projects/<project_name>

# Azure OpenAI Configuration
AZURE_OPENAI_ENDPOINT=https://<account_name>.openai.azure.com/
AZURE_OPENAI_KEY=<your_openai_key>
AZURE_OPENAI_API_KEY=<your_openai_api_key>

# For red team testing (can be same as primary)
AZURE_OPENAI_ENDPOINT_REDTEAM=https://<account_name>.openai.azure.com/
AZURE_OPENAI_DEPLOYMENT=<your_deployment_name>

# API Configuration
AZURE_API_VERSION=2024-10-21

# Azure Resource Configuration
AZURE_SUBSCRIPTION_ID=<your_subscription_id>
AZURE_RESOURCE_GROUP=<your_resource_group>
AZURE_PROJECT_NAME=<your_project_name>

# Optional: Azure AI Search (for search agent features)
AZURE_SEARCH_ENDPOINT=https://<search_service>.search.windows.net
AZURE_SEARCH_KEY=<your_search_key>
AZURE_SEARCH_INDEX=<your_index_name>

# Optional: Email Configuration (for connected agent email functionality)
GOOGLE_EMAIL=<your_gmail_address>
GOOGLE_APP_PASSWORD=<your_gmail_app_password>

# Tracing Configuration (automatically set by the application)
# AZURE_TRACING_GEN_AI_CONTENT_RECORDING_ENABLED=true
GITHUB_PAT_TOKEN=<your_github_pat_token>
```

## Code Steps

- Load necessary libraries:

```
import tempfile
import uuid
import asyncio
import io
import os
import time
import json
from datetime import datetime
from typing import Optional, Dict, Any, List
import glob
from pathlib import Path
import sys

try:
    import numpy as np
    NUMPY_AVAILABLE = True
except ImportError:
    NUMPY_AVAILABLE = False

try:
    import streamlit as st
    STREAMLIT_AVAILABLE = True
except ImportError:
    STREAMLIT_AVAILABLE = False

try:
    from openai import AzureOpenAI
    OPENAI_AVAILABLE = True
except ImportError:
    OPENAI_AVAILABLE = False
    print("Warning: OpenAI not available. Running in test mode.")

try:
    from dotenv import load_dotenv
    load_dotenv()
except ImportError:
    print("Warning: python-dotenv not available.")

# Load environment variables
# Azure OpenAI configuration (replace with your credentials)
AZURE_ENDPOINT = os.getenv("AZURE_OPENAI_ENDPOINT")
AZURE_API_KEY = os.getenv("AZURE_OPENAI_KEY")
WHISPER_DEPLOYMENT_NAME = "whisper"
CHAT_DEPLOYMENT_NAME = os.getenv("AZURE_OPENAI_DEPLOYMENT")
```

- Function to process code files:

```
def codexmini_code_response_api(transcription, instructions=None):
    returntxt = ""
    """Generate a chat response using Azure OpenAI with tool calls."""
    codeclient = AzureOpenAI(  
        base_url = os.getenv("AZURE_OPENAI_ENDPOINT") + "/openai/v1/",  
        api_key= os.getenv("AZURE_OPENAI_KEY"),
        api_version="preview"
        )
    
    
    prompt = f"""
    You are a helpful assistant that converts code from one language to another.
    Follow the user's instructions carefully: {instructions}
    
    Please return your response in JSON format with the following structure:
    {{
        "code": "converted code here",
        "source_language": "detected source language",
        "target_language": "target language"
    }}

    User Query:
    {transcription}
    """
    
    messages = [
        {"role": "system", "content": "You are a helpful code conversion assistant. Always return valid JSON."},
        {"role": "user", "content": prompt}
    ]

    try:
        # Use the regular chat completions API since codex-mini might not be available
        response = codeclient.responses.create(
            model="codex-mini", # replace with your model deployment name 
            input=messages,
            max_output_tokens= 2500,
            instructions="Convert the following code to Python.",
        )
        # returntxt = response.choices[0].message.content.strip()
        returntxt = response.output_text
        print('Response:', returntxt)
        return returntxt

    except Exception as e:
        print(f"An error occurred: {e}")
        # Return a proper JSON error response
        error_response = {
            "code": f"Error: {str(e)}",
            "source_language": "unknown",
            "target_language": "unknown"
        }
        return json.dumps(error_response)

def parse_output(json_input):
    # Handle None input
    if json_input is None:
        return "Error: No response received", "unknown", "unknown"
    
    try:
        # If input is a string, parse it as JSON
        if isinstance(json_input, str):
            # Try to extract JSON from the response if it's embedded in text
            import re
            json_match = re.search(r'\{.*\}', json_input, re.DOTALL)
            if json_match:
                json_str = json_match.group()
                data = json.loads(json_str)
            else:
                # If no JSON found, return the string as code
                return json_input, "unknown", "unknown"
        else:
            data = json_input

        # Extract required fields with defaults
        code = data.get("code", json_input if isinstance(json_input, str) else "")
        source = data.get("source_language", "unknown")
        target = data.get("target_language", "unknown")
        return code, source, target
        
    except json.JSONDecodeError as e:
        print(f"JSON parsing error: {e}")
        # Return the raw input as code if JSON parsing fails
        return str(json_input), "unknown", "unknown"
    except Exception as e:
        print(f"Unexpected error in parse_output: {e}")
        return f"Error: {str(e)}", "unknown", "unknown"
```

- Now create a Streamlit app to upload code files and process them:

```python
def codeconvmain():
    if not STREAMLIT_AVAILABLE:
        print("Streamlit not available. Use --batch mode for CLI conversion.")
        return
        
    st.set_page_config(
        page_title="Code Conversion Tool",
        page_icon="�",
        layout="wide",
        initial_sidebar_state="expanded"
    )
    # Instruction input with card styling
    st.markdown("### 📝 Conversion Instructions")
    instruction = st.text_input(
        "Enter the instruction for code conversion",
        value="Convert the following code to Python",
        help="Specify what language to convert to and any special requirements"
    )
    st.markdown('</div>', unsafe_allow_html=True)

    col1, col2 = st.columns([1, 1], gap="large")

    with col1:
        st.markdown("### 📥 Input Code")
        if st.session_state.source_lang != "unknown":
            st.markdown(f'<span class="language-badge">Source: {st.session_state.source_lang}</span>', unsafe_allow_html=True)
        input_code = st.text_area("Enter your code here", height=300, placeholder="Paste your code here...")
        st.markdown('</div>', unsafe_allow_html=True)

    with col2:
        st.markdown("### 📤 Converted Code")
        
        # Language badges and conversion arrow
        if st.session_state.source_lang != "unknown" and st.session_state.target_lang != "unknown":
            col_lang1, col_arrow, col_lang2 = st.columns([1, 0.5, 1])
            with col_lang1:
                st.markdown(f'<span class="language-badge">From: {st.session_state.source_lang}</span>', unsafe_allow_html=True)
            with col_arrow:
                st.markdown('<div class="conversion-arrow">→</div>', unsafe_allow_html=True)
            with col_lang2:
                st.markdown(f'<span class="language-badge">To: {st.session_state.target_lang}</span>', unsafe_allow_html=True)
        
        # Convert button
        convert_clicked = st.button("🔄 Convert Code", use_container_width=True)
        
        if convert_clicked:
            if not input_code.strip():
                st.warning("⚠️ Please enter some code to convert!")
            else:
                with st.spinner("🔄 Converting code... Please wait"):
                    try:
                        # output_code = codexmini_code_response(input_code, instruction)
                        output_code = codexmini_code_response_api(input_code, instruction)
                        code, source, target = parse_output(output_code)
                        
                        # Update session state
                        st.session_state.output_code = code.strip()
                        st.session_state.source_lang = source
                        st.session_state.target_lang = target
                        
                        # Success message
                        st.success("✅ Code converted successfully!")
                        
                        # Display language info
                        if source != "unknown" or target != "unknown":
                            st.info(f"📊 Detected: {source} → {target}")
                        
                        # Display converted code
                        st.code(code, language=target.lower() if target != "unknown" else None)
                        
                    except Exception as e:
                        st.error(f"❌ Error during conversion: {str(e)}")
        
        # Display existing code if available
        elif st.session_state.output_code:
            st.code(st.session_state.output_code, language=st.session_state.target_lang.lower() if st.session_state.target_lang != "unknown" else None)
        
        st.markdown('</div>', unsafe_allow_html=True)

    # Footer with additional info
    st.markdown("---")
```

- now create the main function to run the Streamlit app:

```python
if __name__ == "__main__":
    run_app()
```

- run the Streamlit app:

```bash
streamlit run app.py
```

- Copy and paste your code into the input area
- Click the "Convert Code" button
- Wait for the conversion to complete
- Check the converted code in the output area