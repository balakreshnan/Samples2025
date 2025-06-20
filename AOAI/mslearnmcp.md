# Voice Enabled Apps with Azure OpenAI

## Introduction

- Create a voice first interface to get Microsoft Learn content.
- Microsoft Learn is MCP enabled.
- Using the Response API, you can create a voice-enabled application that interacts with Microsoft Learn content.
- Display the text of the response in a web page.
- Play back audio as reposnse for conversation.
- Use the Azure OpenAI Service to generate responses based on user queries.

## Prerequisites

- Azure Subscription
- Azure AI Foundry Project resource
- Pick East US 2 as region
- Need gpt 4.1, gpt-4o-mini-tts models
- Need whisper model for speech to text
- Azure Storage for AI foundry services
- Azure Monitor for observability
- Using Streamlit web app as a front end.

## Steps to Create the Application

- Create a python 3.12 virtual environment.
- Install the required packages:

```
openai
streamlit
python-dotenv
gtts
```

- Here is a sample env file:

```
PROJECT_ENDPOINT=""
MODEL_ENDPOINT=""
MODEL_API_KEY=""
MODEL_DEPLOYMENT_NAME=""
AZURE_AI_PROJECT=""
AZURE_SUBSCRIPTION_ID=""
AZURE_RESOURCE_GROUP=""
AZURE_PROJECT_NAME=""
AZURE_OPENAI_KEY=""
AZURE_OPENAI_ENDPOINT=""
AZURE_OPENAI_DEPLOYMENT=""
AZURE_OPENAI_ENDPOINT_TTS=""
AZURE_OPENAI_KEY_TTS=""
```

- My project is in sweden central region and doesn't have gpt-4o-mini-tts model, hence i have tts in another region.
- Create a Streamlit app file `app.py`:
- load the libraries and environment variables:

```
import json
import requests
import streamlit as st
from openai import AzureOpenAI

import base64
import os
from gtts import gTTS
import tempfile
import uuid
from dotenv import load_dotenv

# Load environment variables
load_dotenv()
```

- Load the environment variables and set up the Azure OpenAI client:

```
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

- Here are the functions to handle speech to text, text to speech, and chat completion:

```
def save_audio_file(audio_data, extension="wav"):
    """Save audio bytes to a temporary file."""
    temp_file = os.path.join(tempfile.gettempdir(), f"audio_{uuid.uuid4()}.{extension}")
    with open(temp_file, "wb") as f:
        f.write(audio_data)
    return temp_file

def transcribe_audio(audio_file_path):
    """Transcribe audio using Azure OpenAI Whisper."""
    with open(audio_file_path, "rb") as audio_file:
        response = client.audio.transcriptions.create(
            file=audio_file,
            model=WHISPER_DEPLOYMENT_NAME,
            response_format="text"
        )
    return response

def generate_audio_response(text):
    """Generate audio response using gTTS."""
    tts = gTTS(text=text, lang="en")
    temp_file = os.path.join(tempfile.gettempdir(), f"response_{uuid.uuid4()}.mp3")
    tts.save(temp_file)
    return temp_file

def generate_audio_response_gpt(text):
    """Generate audio response using gTTS."""
    # tts = gTTS(text=text, lang="en")
    url = os.getenv("AZURE_OPENAI_ENDPOINT_TTS")  
  
    headers = {  
        "Content-Type": "application/json",  
        "Authorization": f"Bearer {os.environ['AZURE_OPENAI_KEY_TTS']}"  
    }  
    
    data = {  
        "model": "gpt-4o-mini-tts",  
        "input": text,  
        "voice": "alloy"  
    }  
    
    response = requests.post(url, headers=headers, json=data)  
    
    print(response.status_code)  
    # print(response.content)  # audio bytes or error message 
    temp_file = os.path.join(tempfile.gettempdir(), f"response_{uuid.uuid4()}.mp3")
    # response.content.save(temp_file)
    if response.status_code == 200:  
        with open(temp_file, "wb") as f:  
            f.write(response.content)  
        print("MP3 file saved successfully.")  
    else:  
        print(f"Error: {response.status_code}\n{response.text}")
    return temp_file

def retrieve_relevant_content(query, json_data):
    """Retrieve relevant content from JSON data based on query keywords."""
    try:
        data = json.loads(json_data)
        query_words = query.lower().split()
        results = []
        
        if isinstance(data, dict):
            for key, value in data.items():
                content = f"{key}: {value}".lower()
                if any(word in content for word in query_words):
                    results.append(f"{key}: {value}")
        elif isinstance(data, list):
            for item in data:
                content = str(item).lower()
                if any(word in content for word in query_words):
                    results.append(str(item))
        
        return "\n".join(results) if results else "No relevant information found."
    except json.JSONDecodeError:
        return "Invalid JSON format provided."

def msft_generate_chat_response(transcription, context):
    """Generate a chat response using Azure OpenAI with tool calls."""
    returntxt = ""
    tools = [
        {
            "type": "function",
            "function": {
                "name": "mcp_tool",
                "description": "Queries the Microsoft Cloud DOcumentation information.",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "query": {
                            "type": "string",
                            "description": "The query to send to the MCP API."
                        }
                    },
                    "required": ["query"]
                }
            }
        }
    ]

    prompt = f"""
    You are a helpful assistant. Use the following context and tools to answer the user's query.
    If the context or tools are not relevant, provide a general response based on the query.
    Only respond with the tool call.
    Ask for followup until you get the right information. Probe the user for more details if necessary.
    If the context is not relevant, provide a general response based on the query.
    Be positive and encouraging in your response. Ignore any negative or irrelevant information.
    please ignore any questions that are not related to learning. 
    DOn't get annoyed or frustrated. if user asks probing questions, please politely ignore them.
    Provide sources and citations for your responses.

    User Query:
    {transcription}
    
    Response:
    """
    messages = [
        {"role": "system", "content": "You are a helpful assistant with access to the MCP API."},
        {"role": "user", "content": prompt}
    ]

    mcpclient = AzureOpenAI(  
        base_url = os.getenv("AZURE_OPENAI_ENDPOINT") + "/openai/v1/",  
        api_key= os.getenv("AZURE_OPENAI_KEY"),
        api_version="preview"
        )

    response = mcpclient.responses.create(
        model=CHAT_DEPLOYMENT_NAME, # replace with your model deployment name 
        tools=[
            {
                "type": "mcp",
                "server_label": "MicrosoftLearn",
                "server_url": "https://learn.microsoft.com/api/mcp",
                "require_approval": "never"
            },
        ],
        input=transcription,
        max_output_tokens= 1500,
        instructions="Generate a response using the MCP API tool.",
    )
    # returntxt = response.choices[0].message.content.strip()
    retturntxt = response.output_text
    print(f"Response: {retturntxt}")
        
    return retturntxt, None
```

- Now create the main streemlit UI:

```
def main():
    # st.title("Voice Chat with RAG (Azure OpenAI)")
    # Use the wide layout
    st.set_page_config(
        page_title="Microsoft Learn UI",
        layout="wide"  # 'centered' is default; use 'wide' for full page width
    )
    st.title("Microsoft Learn UI Audio Interface")

    # Initialize session state for chat history
    if "messages" not in st.session_state:
        st.session_state.messages = []

    json_input = ""

    # Display chat history
    for message in st.session_state.messages:
        with st.chat_message(message["role"]):
            st.markdown(message["content"])
            if message.get("audio"):
                audio_base64 = base64.b64encode(message["audio"]).decode()
                audio_html = f"""
                <audio controls>
                    <source src="data:audio/mp3;base64,{audio_base64}" type="audio/mp3">
                </audio>
                """
                st.markdown(audio_html, unsafe_allow_html=True)

    # Audio input
    audio_value = st.audio_input("Record your voice message")

    if audio_value:
        # Process user input
        with st.chat_message("user"):
            st.audio(audio_value)
            with st.spinner("Transcribing audio..."):
                audio_file_path = save_audio_file(audio_value.getvalue())
                transcription = transcribe_audio(audio_file_path)
                st.markdown(transcription)
                
                # Save user message
                st.session_state.messages.append({"role": "user", "content": transcription})

        # Process assistant response
        with st.chat_message("assistant"):
            with st.spinner("Generating response..."):
                # Retrieve relevant content from JSON
                context = retrieve_relevant_content(transcription, json_input)
                response_text, mcp_result = msft_generate_chat_response(transcription, context)
                # response_audio_path = generate_audio_response(response_text)
                response_audio_path = generate_audio_response_gpt(response_text)
                
                st.markdown(response_text)
                with open(response_audio_path, "rb") as f:
                    audio_bytes = f.read()
                    audio_base64 = base64.b64encode(audio_bytes).decode()
                    audio_html = f"""
                    <audio controls autoplay>
                        <source src="data:audio/mp3;base64,{audio_base64}" type="audio/mp3">
                    </audio>
                    """
                    st.markdown(audio_html, unsafe_allow_html=True)
                
                # Save assistant message
                st.session_state.messages.append({
                    "role": "assistant",
                    "content": response_text,
                    "audio": audio_bytes
                })

            # Clean up temporary files
            os.remove(audio_file_path)
            os.remove(response_audio_path)
```

- Create the main function to run the Streamlit app:

```
if __name__ == "__main__":
    main()
```

- Run the Streamlit app:

```
streamlit run app.py
```

- Open the app in your browser at `http://localhost:8501`.
- You can now interact with the application by recording your voice, and it will transcribe your speech, generate a response using Azure OpenAI, and play back the audio response.
- Check the response coming from Microsoft Learn MCP and also the audio response generated by gpt-4o-mini-tts model.

![info](https://github.com/balakreshnan/Samples2025/blob/main/AOAI/images/mslearnmcp-1.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AOAI/images/mslearnmcp-2.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AOAI/images/mslearnmcp-3.jpg 'RagChat')

## Conclusion

- Now we can create a voice-enabled application that interacts with Microsoft Learn content using Azure OpenAI.
- We can look at designing new ways of voice interaction devices.
- Can it be wearable or a mobile app? or a smart speaker? or a smart display? or laptop or desktop? or a kiosk?
- Have to work on response being natural rather than reading the text output.
- More work is needed to improve the user experience and make the interaction more engaging.