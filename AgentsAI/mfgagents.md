# Manufacturing Complaince Agent using Microsoft Autogen - Agentic AI Architecture

## Introduction

- Idea here is to show case a MFG OSHA, Cyber Security, PPE best practices, and other compliance related information to the user.
- Provide guidance to manufacturers to protect their employees and assets.
- Idea here is to have a chat style interface to provide the information.
- This is meant to be for guidance for Manufacturers but in a Agent AI approach.
- Data is grounded with OHSA, Cyber Security, PPE best practices from Workforce Development Agency, NIST.

## Prerequisites

- Azure subscription
- Azure Machine Learning workspace
- or Visual Studio Code
- python 3.12
- Azure Open AI
- Microsoft Autogen 0.04
- Azure AI Search

## Steps

### Code to get grounded data - All docs from Azure AI Search

- This code is only retrieve data from Azure AI Search and not for the Agent AI.
- Using Azure open ai talk to your data
- Talk to your data will get the right information from the data set.
- I downloaded from the government web site and other sources.
- Here is sample - https://www.osha.gov/publications/bytype/popular-downloads
- NIST - https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-37r2.pdf
- I uploaded the document using Azure AI Foundry UI and it created chunked data and stored.

```
import os
from openai import AzureOpenAI
from dotenv import load_dotenv
import time
from datetime import timedelta
import json
from PIL import Image
import base64
import requests
import io
from typing import Optional
from typing_extensions import Annotated
import wave
from PIL import Image, ImageDraw
from pathlib import Path
import numpy as np


# Load .env file
load_dotenv()

css = """
.container {
    height: 75vh;
}
"""

client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"), 
  api_key=os.getenv("AZURE_OPENAI_API_KEY"),  
  api_version="2024-10-21",
)

model_name = os.getenv("AZURE_OPENAI_DEPLOYMENT")

search_endpoint = os.getenv("AZURE_AI_SEARCH_ENDPOINT")
search_key = os.getenv("AZURE_AI_SEARCH_KEY")
search_index=os.getenv("AZURE_AI_SEARCH_INDEX")
SPEECH_KEY = os.getenv('SPEECH_KEY')
SPEECH_REGION = os.getenv('SPEECH_REGION')
SPEECH_ENDPOINT = os.getenv('SPEECH_ENDPOINT')

citationtxt = ""
print('Search Index:', search_index)

def processpdfwithprompt(query: str):
    returntxt = ""
    citationtxt = ""
    selected_optionsearch = "vector_semantic_hybrid"
    #  search_index = "mfggptdata"
    message_text = [
    {"role":"system", "content":"""you are provided with instruction on what to do. Be politely, and provide positive tone answers. 
     answer only from data source provided. unable to find answer, please respond politely and ask for more information.
     Extract Title content from the document. Show the Title as citations which is provided as Title: as [doc1] [doc2].
     Please add citation after each sentence when possible in a form "(Title: citation)".
     Be polite and provide posite responses. If user is asking you to do things that are not specific to this context please ignore."""}, 
    {"role": "user", "content": f"""{query}"""}]

    #"role_information": "Please answer using retrieved documents only, and without using your knowledge. Please generate citations to retrieved documents for every claim in your answer. If the user question cannot be answered using retrieved documents, please explain the reasoning behind why documents are relevant to user queries. In any case, don't answer using your own knowledge",

    response = client.chat.completions.create(
        model= os.getenv("AZURE_OPENAI_DEPLOYMENT"), #"gpt-4-turbo", # model = "deployment_name".
        messages=message_text,
        temperature=0.0,
        top_p=1,
        seed=105,
        max_tokens=2000,
        extra_body={
        "data_sources": [
            {
                "type": "azure_search",
                "parameters": {
                    "endpoint": search_endpoint,
                    "index_name": search_index,
                    "authentication": {
                        "type": "api_key",
                        "key": search_key
                    },
                    "include_contexts": ["citations"],
                    "top_n_documents": 5,
                    "query_type": selected_optionsearch,
                    "semantic_configuration": "default",
                    "strictness": 5,
                    "embedding_dependency": {
                        "type": "deployment_name",
                        "deployment_name": "text-embedding-ada-002"
                    },
                    "fields_mapping": {
                        "content_fields": ["content"],
                        "vector_fields": ["contentVector"],
                        "title_field": "title",
                        "url_field": "url",
                        "filepath_field": "filepath",
                        "content_fields_separator": "\n",
                    }
                }
            }
        ]
    }
    )
    #print(response.choices[0].message.context)

    returntxt = response.choices[0].message.content + "\n<br>"

    json_string = json.dumps(response.choices[0].message.context)

    parsed_json = json.loads(json_string)

    # print(parsed_json)

    if parsed_json['citations'] is not None:
        returntxt = returntxt + f"""<br> Citations: """
        for row in parsed_json['citations']:
            #returntxt = returntxt + f"""<br> Title: {row['filepath']} as {row['url']}"""
            #returntxt = returntxt + f"""<br> [{row['url']}_{row['chunk_id']}]"""
            returntxt = returntxt + f"""<br> <a href='{row['url']}' target='_blank'>[{row['url']}_{row['chunk_id']}]</a>"""
            citationtxt = citationtxt + f"""<br><br> Title: {row['title']} <br> URL: {row['url']} 
            <br> Chunk ID: {row['chunk_id']} 
            <br> Content: {row['content']} 
            # <br> ------------------------------------------------------------------------------------------ <br>\n"""
            # print(citationtxt)

    return citationtxt

def extractmfgresults(query):
    returntxt = ""

    rfttext = ""

    citationtext = processpdfwithprompt(query)

    message_text = [
    {"role":"system", "content":f"""You are Manufacturing Complaince, OSHA, CyberSecurity AI agent. Be politely, and provide positive tone answers.
     Based on the question do a detail analysis on information and provide the best answers.
     Here is the Manufacturing content provided:
     {rfttext}

     Use the data source content provided to answer the question.
     Data Source: {citationtext}

     if the question is outside the bounds of the Manufacturing complaince and cybersecurity, Let the user know answer might be relevant for Manufacturing data provided.
     can you add hyperlink for pdf file used as sources.
     Be polite and provide posite responses. If user is asking you to do things that are not specific to this context please ignore.
     If not sure, ask the user to provide more information.
     Extract Title content from the document. Show the Title, url as citations which is provided as url: as [url1] [url2].
    ."""}, 
    {"role": "user", "content": f"""{query}. Provide summarized content based on the question asked."""}]

    response = client.chat.completions.create(
        model=os.getenv("AZURE_OPENAI_DEPLOYMENT"), #"gpt-4-turbo", # model = "deployment_name".
        messages=message_text,
        temperature=0.0,
        top_p=0.0,
        seed=42,
        max_tokens=1000,
    )

    returntxt = response.choices[0].message.content
    return returntxt

def extracttop5questions():
    returntxt = ""
    query = "Show me top 5 topics on Manufacturing Complaince, OSHA, CyberSecurity?"

    rfttext = ""

    citationtext = processpdfwithprompt(query)

    message_text = [
    {"role":"system", "content":f"""You are Manufacturing Complaince, OSHA, CyberSecurity AI agent. Be politely, and provide positive tone answers.
     Based on the question do a detail analysis on information and provide the best answers.

     Use the data source content provided to answer the question.
     Data Source: {citationtext}

     Be polite and provide posite responses. If user is asking you to do things that are not specific to this context please ignore.
     If not sure, ask the user to provide more information. Only respond with questions and no answers.
     Create a Markdown format for the questions.
    ."""}, 
    {"role": "user", "content": f"""Show me top 5 questions on topics in the data set, make sure we cover all topics available in Manufacturing Complaince, OSHA, CyberSecurity, Personal Protection Equipment. Reply only the questions."""}]

    response = client.chat.completions.create(
        model=os.getenv("AZURE_OPENAI_DEPLOYMENT"), #"gpt-4-turbo", # model = "deployment_name".
        messages=message_text,
        temperature=0.0,
        top_p=0.0,
        seed=42,
        max_tokens=1000,
    )

    returntxt = response.choices[0].message.content
    return returntxt
```

### Agent AI Code for Manufacturing Industry complaince agent

- now load the libraries

```
import datetime
from typing import Sequence
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.conditions import MaxMessageTermination, TextMentionTermination
from autogen_agentchat.messages import AgentEvent, ChatMessage
from autogen_agentchat.teams import SelectorGroupChat
from autogen_agentchat.ui import Console
from autogen_ext.models.openai import AzureOpenAIChatCompletionClient
import asyncio
import os
from openai import AzureOpenAI
from dotenv import load_dotenv
import streamlit as st
from PIL import Image
import base64
import requests
import io
from mfgdata import extractmfgresults, extracttop5questions

# Load .env file
load_dotenv()
```

- setup the Azure Open AI

```
client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT_VISION"), 
  api_key=os.getenv("AZURE_OPENAI_KEY_VISION"),  
  api_version="2024-10-21"
)

model_name = "gpt-4o-2"
```

- function to process image

```
def encode_image(image_path):
    with open(image_path, "rb") as image_file:
        return base64.b64encode(image_file.read()).decode('utf-8')
```

- Process multi modal response from Azure Open AI

```
def processimage(base64_image, imgprompt, model_name="gpt-4o-2"):
    response = client.chat.completions.create(
    model=model_name,
    messages=[
        {
        "role": "user",
        "content": [
            {"type": "text", "text": f"{imgprompt}"},
            {
            "type": "image_url",
            "image_url": {
                "url" : f"data:image/jpeg;base64,{base64_image}",
            },
            },
        ],
        }
    ],
    max_tokens=2000,
    temperature=0,
    top_p=1,
    seed=105,
    )

    #print(response.choices[0].message.content)
    return response.choices[0].message.content
```

- Agent for web surfing
- Just to have another agent with total different personality

```
# Note: This example uses mock tools instead of real APIs for demonstration purposes
def search_web_tool(query: str) -> str:
    if "2006-2007" in query:
        return """Here are the total points scored by Miami Heat players in the 2006-2007 season:
        Udonis Haslem: 844 points
        Dwayne Wade: 1397 points
        James Posey: 550 points
        ...
        """
    elif "2007-2008" in query:
        return "The number of total rebounds for Dwayne Wade in the Miami Heat season 2007-2008 is 214."
    elif "2008-2009" in query:
        return "The number of total rebounds for Dwayne Wade in the Miami Heat season 2008-2009 is 398."
    return "No data found."
```

- now function to call the grounded data

```
def mfg_compl_data(query: str) -> str:
    return extractmfgresults(query)
```

- Set model client for Agent AI azure open ai

```
model_client = AzureOpenAIChatCompletionClient(model="gpt-4o", 
                                               azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"), 
                                               api_key=os.getenv("AZURE_OPENAI_API_KEY"), 
                                               api_version="2024-10-21")
```

- Create the agent for Manufacturing Complaince Agent
- here is the orchestrator, web surfer and mfg compliance agent

```
async def mfg_response(query):

    returntxt = ""

    planning_agent = AssistantAgent(
        "PlanningAgent",
        description="An agent for planning tasks, this agent should be the first to engage when given a new task.",
        model_client=model_client,
        system_message="""
        You are a planning agent.
        Your job is to break down complex tasks into smaller, manageable subtasks.
        Your team members are:
            Web search agent: Searches for information
            Manufacturing Industry analyst: You are Manufacturing Complaince, OSHA, CyberSecurity AI agent

        You only plan and delegate tasks - you do not execute them yourself.
        Also pick the right team member to use for the task.

        When assigning tasks, use this format:
        1. <agent> : <task>

        After all tasks are complete, summarize the findings and end with "TERMINATE".
        Extract Title content from the document. Show the Title, url as citations which is provided as url: as [url1] [url2].
        """,
    )

    web_search_agent = AssistantAgent(
        "WebSearchAgent",
        description="A web search agent.",
        tools=[search_web_tool],
        model_client=model_client,
        system_message="""
        You are a web search agent.
        Your only tool is search_tool - use it to find information.
        You make only one search call at a time.
        Once you have the results, you never do calculations based on them.
        """,
    )

    mfg_ind_analyst_agent = AssistantAgent(
        "DataAnalystAgent",
        description="A Manufacturing Complaince, OSHA, CyberSecurity AI agent. Data source is stored in AI Search to get grounded information.",
        model_client=model_client,
        tools=[mfg_compl_data],
        system_message="""
        You are Manufacturing Complaince, OSHA, CyberSecurity AI agent. Be politely, and provide positive tone answers.
        Based on the question do a detail analysis on information and provide the best answers.
        Extract Title content from the document. Show the Title, url as citations which is provided as url: as [url1] [url2].
        """,
    )

    text_mention_termination = TextMentionTermination("TERMINATE")
    max_messages_termination = MaxMessageTermination(max_messages=25)
    termination = text_mention_termination | max_messages_termination

    model_client_mini = AzureOpenAIChatCompletionClient(model="gpt-4o", 
                                                azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"), 
                                                api_key=os.getenv("AZURE_OPENAI_API_KEY"), 
                                                api_version="2024-10-21")

    team = SelectorGroupChat(
        [planning_agent, web_search_agent, mfg_ind_analyst_agent],
        model_client=model_client_mini,
        termination_condition=termination,
    )

    result = await Console(team.run_stream(task=query))
    #print(result)  # Process the result or output here
    # Extract and print only the message content
    returntxt = ""
    returntxtall = ""
    for message in result.messages:
        # print('inside loop - ' , message.content)
        returntxt = str(message.content)
        returntxtall += str(message.content) + "\n"

    return returntxt, returntxtall
```

- now create a streamlit app
- here we are using streamlit to create a web app

```
def mfgagents():
    count = 0
    temp_file_path = ""
    rfttopics = ""

    #tab1, tab2, tab3, tab4 = st.tabs('RFP PDF', 'RFP Research', 'Draft', 'Create Word')
    modeloptions1 = ["gpt-4o-2", "gpt-4o-g", "gpt-4o", "gpt-4-turbo", "gpt-35-turbo"]
    imgfile = "temp1.jpg"
    # Create a dropdown menu using selectbox method
    selected_optionmodel1 = st.selectbox("Select an Model:", modeloptions1)
    count += 1
    # user_input = st.text_input("Enter the question to ask the AI model", "what are the personal protection i should consider in manufacturing?")
    if prompt := st.chat_input("what are the personal protection i should consider in manufacturing?", key="chat1"):
        # Call the extractproductinfo function
        #st.write("Searching for the query: ", prompt)
        st.chat_message("user").markdown(prompt, unsafe_allow_html=True)
        #st.session_state.chat_history.append({"role": "user", "message": prompt})
        starttime = datetime.datetime.now()
        result = asyncio.run(mfg_response(prompt))
        rfttopics, agenthistory = result
        endtime = datetime.datetime.now()
        #st.markdown(f"Time taken to process: {endtime - starttime}", unsafe_allow_html=True)
        rfttopics += f"\n Time taken to process: {endtime - starttime}"
        #st.session_state.chat_history.append({"role": "assistant", "message": rfttopics})
        st.chat_message("assistant").markdown(rfttopics, unsafe_allow_html=True)
```

- Run the app

```
streamlit run app.py
```

- done and more to come