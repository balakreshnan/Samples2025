# Azure Open AI 4.1 models in Azure AI Foundry

## Introduction

- Azure Open AI 4.1 models are now available in Azure AI Foundry.
- Use the 4.1 model to create source code and run it in Azure AI Foundry.
- Sample to run using Visual Studio Code and Azure AI Foundry.
- This is just a test to show case how to consume the 4.1 model in Azure AI Foundry.

## Pre-requisites

- Azure Subscription
- Azure AI Foundry
- Azure Open AI 4.1 model
- Visual Studio Code
- python 3.12 or later
- Install Open AI SDK for Python

## Code

- Go to Azure AI Foundry, deploy the model from Model catalog.
- Set the environment variable for Azure Open AI 4.1 model
- Make sure Azure open ai endpoint points to Azure AI Foundry URL

- here is the code to run the 4.1 model in Azure AI Foundry.

```python
import os
from openai import AzureOpenAI

from dotenv import load_dotenv
load_dotenv()

endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
model_name = "gpt-4.1"
deployment = "gpt-4.1"

def main(query: str) -> str:
    subscription_key = os.getenv("AZURE_OPENAI_API_KEY")
    api_version = "2024-12-01-preview"
    returnstr = ""
    print(f"Endpoint: {endpoint}")

    client = AzureOpenAI(
        api_version=api_version,
        azure_endpoint=endpoint,
        api_key=subscription_key,
    )

    response = client.chat.completions.create(
        messages=[
            {
                "role": "system",
                "content": "You are a helpful assistant.",
            },
            {
                "role": "user",
                "content": query,
            }
        ],
        max_completion_tokens=8000,
        temperature=1.0,
        top_p=1.0,
        frequency_penalty=0.0,
        presence_penalty=0.0,
        model=deployment
    )

    print(response.choices[0].message.content)
    returnstr = response.choices[0].message.content
    return returnstr

if __name__ == "__main__":
    query = "Create a python code to build a web app to upload a image and use Azure open AI models to analyze the image and return the result."

    result = main(query)
    print(f"Result: {query}")
```

- run the code using the command line or Visual Studio Code.
- Run the app

```
python app.py
```

- here is the output of the code.

```
Sure! To build a web app that uploads an image and analyzes it using Azure OpenAI models, you can use:

**Frontend/Backend:** Streamlit (simple, Python-based web apps)
**Image Analysis:** Azure OpenAI models currently only analyze text and text/image tasks (e.g., GPT-4 Vision).
**API**: Use the Azure OpenAI "GPT-4 Vision" API to send the image and get the analysis.

## Prerequisites

- Azure OpenAI resource with GPT-4 Vision enabled ([Docs](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models#gpt-4-vision))
- Your endpoint, API key, and deployment name.
- Install **requests** and **streamlit**:

bash
pip install streamlit requests


## Python Code

Create a file named `app.py`:

python
import streamlit as st
import requests
import base64
from io import BytesIO

# --- Azure settings ---
AZURE_OPENAI_ENDPOINT = "https://<YOUR-RESOURCE-NAME>.openai.azure.com/"
AZURE_OPENAI_KEY = "<YOUR-AZURE-OPENAI-KEY>"
AZURE_OPENAI_DEPLOYMENT = "<YOUR-DEPLOYMENT-NAME>"  # typically 'gpt-4-vision'

# Streamlit UI
st.title("Azure OpenAI Vision Image Analyzer")

uploaded_file = st.file_uploader("Upload an image...", type=["jpg", "jpeg", "png"])
prompt = st.text_area("Prompt", value="Describe the content of this image.")

if uploaded_file is not None:

    # Convert image file to base64
    img_bytes = uploaded_file.read()
    img_base64 = base64.b64encode(img_bytes).decode('utf-8')
    # Prepare image for Azure API payload
    image_dict = {
        "type": "image_url",
        "image_url": {
            "url": f"data:image/png;base64,{img_base64}"
        }
    }

    headers = {
        "api-key": AZURE_OPENAI_KEY,
        "Content-Type": "application/json"
    }

    # Craft the message following Azure's expected schema
    data = {
        "messages": [
            {
                "role": "user",
                "content": [
                    {"type": "text", "text": prompt},
                    image_dict
                ]
            }
        ],
        "max_tokens": 1024
    }

    if st.button("Analyze Image"):
        # Endpoint for Azure OpenAI Chat Completions (GPT-4 Vision)
        api_url = (
            f"{AZURE_OPENAI_ENDPOINT}openai/deployments/"
            f"{AZURE_OPENAI_DEPLOYMENT}/chat/completions?api-version=2024-02-15-preview"
        )

        with st.spinner("Analyzing..."):
            response = requests.post(api_url, headers=headers, json=data)
            if response.status_code == 200:
                result = response.json()
                answer = result["choices"][0]["message"]["content"]
                st.image(BytesIO(img_bytes), caption='Uploaded Image')
                st.markdown("**Analysis:**")
                st.write(answer)
            else:
                st.error(f"Error: {response.status_code} {response.text}")


## Instructions

1. Replace `<YOUR-RESOURCE-NAME>`, `<YOUR-AZURE-OPENAI-KEY>`, and `<YOUR-DEPLOYMENT-NAME>` with your Azure details.
2. Run the app:


streamlit run app.py


3. Upload an image and click **Analyze Image**.

---

### Notes

- If you have issues with base64 URLs, check your deployment and API version (GPT-4 Vision should support base64).
- For large images, consider resizing before sending.
- Azure's GPT-4 Vision is required; normal GPT-3.5 and GPT-4 text models do not analyze images.

Let me know if you need examples using FastAPI or Flask, or if you want to store the analysis results!
Result: Create a python code to build a web app to upload a image and use Azure open AI models to analyze the image and return the result.
```

- More to come.
- next would be using 1 million tokens for context and see the output.