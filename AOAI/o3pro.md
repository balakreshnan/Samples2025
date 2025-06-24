# o3-pro process image and text

## Introduction

- o3-pro is a tool for processing images and text.
- To show how to use the python sdk to send images and text to o3-pro for processing.
- Only for development and testing purposes, not for production use.

## Prerequisites

- Azure Subscription
- Azure AI Foundry in East US 2 region
- Deploy o3-pro in Azure AI Foundry
- Python 3.12 or later
- Visual Studio Code or any other code editor
- Using Streamlit to build the UI

## Steps

- Import the necessary libraries:

```python
import datetime
from io import BytesIO
import json
import requests
import streamlit as st
import streamlit.components.v1 as components
from openai import AzureOpenAI

import base64
import os
from gtts import gTTS
import tempfile
import uuid
from dotenv import load_dotenv
from PIL import Image

# Load environment variables
load_dotenv()
```

- Set up the Azure OpenAI client:

```python
# Azure OpenAI configuration (replace with your credentials)
AZURE_ENDPOINT = os.getenv("AZURE_OPENAI_ENDPOINT")
AZURE_API_KEY = os.getenv("AZURE_OPENAI_KEY")
WHISPER_DEPLOYMENT_NAME = "whisper"
CHAT_DEPLOYMENT_NAME = os.getenv("AZURE_OPENAI_DEPLOYMENT")

# Initialize Azure OpenAI client
client = AzureOpenAI(
    azure_endpoint=AZURE_ENDPOINT,
    api_key=AZURE_API_KEY,
    api_version="2024-06-01"  # Adjust API version as needed
)
```

- Image function to convert image to base64:

```python
# Convert image to base64 for API
def image_to_base64(image):
    buffered = BytesIO()
    image.save(buffered, format="JPEG")
    return base64.b64encode(buffered.getvalue()).decode("utf-8")
```

- Here is the code to process the image and text:

```python
# Analyze image with Azure OpenAI o3 model (using Responses API)
def analyze_with_o3(image, question):

    returntxt = ""
    Starttime = datetime.datetime.now()
    
    try:
        image_base64 = image_to_base64(image)
        # Getting the Base64 string
        # base64_image = encode_image(image_path)

        
        # Initialize Azure OpenAI client
        reasoningclient = AzureOpenAI(
            azure_endpoint=AZURE_ENDPOINT,
            api_key=AZURE_API_KEY,
            api_version="2025-04-01-preview"  # Adjust API version as needed
        )

        response = reasoningclient.responses.create(
            input=[
                {
                    "role": "user",
                    "content": [
                        { "type": "input_text", "text": f"{question}" },
                        {
                            "type": "input_image",
                            "image_url": f"data:image/jpeg;base64,{image_base64}"
                        }
                    ]
                }
            ],
            model="o3-pro", # "o4-mini", # replace with model deployment name
            reasoning={
                "effort": "medium",
                "summary": "detailed" # auto, concise, or detailed (currently only supported with o4-mini and o3)
            },
            max_output_tokens=4000,
        )

        print(response.output_text)
        Endtime = datetime.datetime.now()
        returntxt = response.output_text
        # returntxt = returntxt.replace("\n", "<br>")  # Replace newlines with HTML line breaks for better display in Streamlit
        returntxt += f"<br><br>\n\nAnalysis completed in {Endtime - Starttime} seconds."

        return returntxt # response.choices[0].message.content
    except Exception as e:
        return f"Error analyzing image with o3 model: {str(e)}"
```

- Streamlit UI to upload image and ask question:

```python
# Streamlit app
def main():
    st.set_page_config(
        page_title="Technical Drawing Analysis",
        layout="wide"  # 'centered' is default; use 'wide' for full page width
    )
    st.title("Image Analysis with Azure GPT-4.1 and o3 Model")
    
    # Image upload
    uploaded_file = st.file_uploader("Upload an image", type=["jpg", "jpeg", "png"])
    if uploaded_file:
        image = Image.open(uploaded_file)
        st.image(image, caption="Uploaded Image", use_container_width=True)
        
        # Question input
        question = st.text_input("Ask a question about the image:", value="Extract materials, quantities, and dimensions from the image as JSON.")
        
        # Model selection
        model_choice = st.selectbox("Choose analysis model:", ["Azure GPT-4.1 Vision", "o3 Model"])
        
        if st.button("Analyze") and question:
            with st.spinner("Analyzing...", show_time=True):
                if model_choice == "Azure GPT-4.1 Vision":
                    result = analyze_with_azure(image, question)
                else:
                    result = analyze_with_o3(image, question)
                st.subheader("Analysis Result")
                st.write(result)

- now the main function to run the Streamlit app:

```python
if __name__ == "__main__":
    main()
```

- Run the streamlit app:

```bash
streamlit run app.py
```

- Upload an image and ask a question
- Type in extract some information from the image
- Wait for it to process

![info](https://github.com/balakreshnan/Samples2025/blob/main/AOAI/images/techdrawpartsextract-1.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AOAI/images/techdrawpartsextract-2.jpg 'RagChat')

## Conclusion

- You have successfully created a Streamlit app that allows users to upload images and ask questions about them.
- The app uses Azure OpenAI's o3-pro model to analyze the images and provide detailed responses.
- Python code to process images and text with Azure OpenAI's o3-pro model.