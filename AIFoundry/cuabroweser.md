# Computer Use Agent (CUA) Browser using OpenAI SDK

## Introduction

- Using the Computer Use Agent (CUA) to interact with a browser.
- Ability to use Responses API to invoke computer use agents.
- Using Azure OpenAI in Azure AI Foundry.
- Executing using python code to interact with the browser.

## Pre-requisites

- Azure Subscription
- Azure AI Foundry project
- Computer-use-preview model deployed in your Azure AI Foundry project in East US region
- Computer use preview is preview model and is not available in all regions
- Install OpenAI Python SDK
- Using Visual Studio Code or any other IDE that supports Python development
- Python version 3.12
- Install necessary libraries using pip:

## Code 

- Import libraries

```
import tempfile
import uuid
from openai import AzureOpenAI
import streamlit as st
import asyncio
import io
import os
import time
import json
import soundfile as sf
import numpy as np
from datetime import datetime
from typing import Optional, Dict, Any, List
from scipy.signal import resample
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
```

- Load Environment Variables

```
from dotenv import load_dotenv
# Load environment variables
load_dotenv()
```

- load environment variables from .env file

```
# Azure OpenAI configuration (replace with your credentials)
AZURE_ENDPOINT = os.getenv("AZURE_OPENAI_ENDPOINT")
AZURE_API_KEY = os.getenv("AZURE_OPENAI_KEY")
CHAT_DEPLOYMENT_NAME = os.getenv("AZURE_OPENAI_DEPLOYMENT")

# Initialize Azure OpenAI client
client = AzureOpenAI(
    azure_endpoint=AZURE_ENDPOINT,
    api_key=AZURE_API_KEY,
    api_version="2024-06-01"  # Adjust API version as needed
)
```

- Function to take the query and call the computer-use-preview model
- Make sure the API version is "preview" for the computer-use-preview model
- I am using Azure Open AI inside Azure AI Foundry project, so no need to create a Azure Open AI resource separately.

```
def cuarun(query: str) -> str:
    cuaclient = AzureOpenAI(  
        base_url = os.getenv("AZURE_OPENAI_ENDPOINT") + "/openai/v1/",  
        api_key= os.getenv("AZURE_OPENAI_KEY"),
        api_version="preview"
        )

    response = cuaclient.responses.create(
        model="computer-use-preview", # set this to your model deployment name
        tools=[{
            "type": "computer_use_preview",
            "display_width": 1024,
            "display_height": 768,
            "environment": "windows" # other possible values: "mac", "windows", "ubuntu", "browser"
        }],
        input=[
            {
                "role": "user",
                "content": "Can you open microsoft word."
            }
        ],
        truncation="auto",
        max_output_tokens= 1500,
        # instructions="Generate a response using the MCP API tool.",
    )
    # returntxt = response.choices[0].message.content.strip()
    # retturntxt = response.output_text
    # print(f"Response: {retturntxt}")
    print(response.output)
    ## response.output is the previous response from the model
    computer_calls = [item for item in response.output if item.type == "computer_call"]
    if not computer_calls:
        print("No computer call found. Output from model:")
        for item in response.output:
            print(item)

    computer_call = computer_calls[0]
    last_call_id = computer_call.call_id
    action = computer_call.action

    print(f"Last call ID: {last_call_id}")
    print(f"Action: {action}")
```

- Main function to run the code

```
if __name__ == "__main__":
    query = "Check the latest AI news on bing.com."
    cuarun(query)
```

- Run the code in your IDE or terminal

```
python your_script_name.py
```

- See the output in the terminal or IDE console.
- This will execute the CUA model and perform the actions specified in the query.
- Try with different queries to see how the model interacts with the browser.

## Conclusion

- This is just to show case of using the Computer Use Agent (CUA) to interact with a browser.
- Only for learning purpose and not for production use.
- Unless there is new specific model for production use, please do not use this in production.
- Thank you and Have Fun!