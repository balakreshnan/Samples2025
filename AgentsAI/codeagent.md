# Autogen 0.4 - Create and Execute Python Code with Agents

## Overview

- Create a orchestration with agents doing various tasks
- First to create a python script that can be executed by the agent
- Then execute the script and store the ouput
- Based on the previous output create a prediction python code
- Execute the prediction code and store the output
- was tested with autogen 0.4.1 stable version

## Requirements

- autogen 0.4.1
- Azure subscription
- Azure open AI resource
- python 3.12 or above

## Setup

- First create an Azure Open AI resource and get the API key and endpoint
- Create a python environment and install the required packages
- Install autogen 0.4.1
- install other necessary packages

```bash
pip install yfinance matplotlib scikit-learn
```

- I haven't figure out how to install the packages automatically with autogen agents
- i am creating a console app for now
- Now import necessay libraries and set the environment variables

```python
from collections import defaultdict
import datetime
from pathlib import Path
import tempfile
import time
from typing import Sequence
import PyPDF2
from autogen_agentchat.agents import AssistantAgent, UserProxyAgent
from autogen_agentchat.conditions import MaxMessageTermination, TextMentionTermination
from autogen_agentchat.messages import AgentEvent, ChatMessage
from autogen_agentchat.teams import SelectorGroupChat
from autogen_agentchat.ui import Console
from autogen_ext.models.openai import AzureOpenAIChatCompletionClient
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import MagenticOneGroupChat
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_core.code_executor import CodeBlock
from autogen_core import CancellationToken
from autogen_ext.code_executors.local import LocalCommandLineCodeExecutor
from autogen_agentchat.ui import Console
from autogen.code_utils import extract_code
from autogen.code_utils import create_virtual_env
import asyncio
import os
import venv
from bs4 import BeautifulSoup
from openai import AzureOpenAI
from dotenv import load_dotenv
import streamlit as st
from PIL import Image
import base64
import requests
import io

# Load .env file
load_dotenv()
```

- .env file is created for the environment variables
- Now set Azure Open AI client and also client for autogen

```
client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT_VISION"), 
  api_key=os.getenv("AZURE_OPENAI_KEY_VISION"),  
  api_version="2024-10-21"
)

model_name = "gpt-4o-2"

model_client = AzureOpenAIChatCompletionClient(model="gpt-4o", 
                                               azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"), 
                                               api_key=os.getenv("AZURE_OPENAI_API_KEY"), 
                                               api_version="2024-10-21")
```

- My focus is not using Open AI version rather Azure Open AI
- Here is the main file with the below process
- Here is the flow
- First ask the LLM to create a python code to get the last 6 months of Tesla stock price, save as stocks.csv and then print
- i am printing the output to check if the code is correct
- Now extract the code from the output
- Execute the code created by above step. Which is for us to download the dataset to use for data science based prediction
- Now use LLM to create a python code to predict the close price using sklearn using the stocks.csv file as input
- Extract the code from the output
- Execute the code created for prediction using sklearn and print the output
- Also save the predictions to stockspred.csv
- Both csv file should be in the coding folder
- First output is the code and then the output of the code execution, getting the data and save
  
![info](https://github.com/balakreshnan/Samples2025/blob/main/AgentsAI/images/codeagent-2.jpg 'RagChat')

- now create the machine learning code

![info](https://github.com/balakreshnan/Samples2025/blob/main/AgentsAI/images/codeagent-1.jpg 'RagChat')


```
async def main():
    st.set_page_config(page_title="CodeAgent", page_icon=":robot:")
    st.title("CodeAgent :robot:")
    st.write("An AI agent that can read and write code to solve programming problems.")
    work_dir = Path("coding")
    work_dir.mkdir(exist_ok=True)

    code_create = AssistantAgent(
        "CreateCodeAgent",
        description="Create code for the given task.",
        #tools=[bing_search_and_summarize],
        model_client=model_client,
        system_message="""
        Create code for the given task. Create code in python and only respond code as output.
        Also add the necessary packages needed for the code to run.
        If the code needs any data, create a json file with the data and add the link to the json file in the code.
        if there are data pelase save the output as csv file called stocks.csv.
        """,        
    )

    # Generate code using the GPTAssistantAgent
    #code_create.initiate_chat(model_client, message="Write a Python function that prints 'Hello, World!'")
    text_mention_termination = TextMentionTermination("TERMINATE")
    max_messages_termination = MaxMessageTermination(max_messages=25)
    termination = text_mention_termination | max_messages_termination
    team = MagenticOneGroupChat([code_create], 
                                 model_client=model_client,
                                 termination_condition=termination, max_turns=1)
    
    # Extract the generated code
    result = await Console(team.run_stream(task="Write a Python function that to chat last 6 months of Tesla stock price and print as table."))
    last_message = result.messages
    code_blocks = extract_code(last_message[-1].content)

    shell_commands = "pip install yfinance matplotlib"
    python_code = ""
    if code_blocks:
        print("Code blocks extracted:", code_blocks)
        #shell_commands = next(block[1] for block in code_blocks if block[0] == 'sh' or block[0] == 'bash' or block[0] == 'shell')
        #print(shell_commands)
        python_code = next(block[1] for block in code_blocks if block[0] == 'python')
        #print('Python Code only: ', python_code)

    
    try:
        venv_dir = ".venv"
        venv_context = create_virtual_env(venv_dir)

        executor = LocalCommandLineCodeExecutor(virtual_env_context=venv_context, work_dir=work_dir)
        shresult = await executor.execute_code_blocks(
                code_blocks=[
                    CodeBlock(language="shell", code=shell_commands),
                ],
                cancellation_token=CancellationToken(),
            )
        print(shresult.output.strip())
    except Exception as e:
        pass

    local_executor = LocalCommandLineCodeExecutor(work_dir=work_dir)
    result = await local_executor.execute_code_blocks(
            code_blocks=[
                CodeBlock(language="python", code=python_code),
            ],
            cancellation_token=CancellationToken(),
        )
    #print(result)
    print(result.output.strip())

    code_create = AssistantAgent(
        "CreateCodeAgent",
        description="Create code for the given task.",
        #tools=[bing_search_and_summarize],
        model_client=model_client,
        system_message="""
        Create code for stocks.csv to predict close price using lstm.
        Also add the necessary packages needed for the code to run.
        If the code needs any data, create a json file with the data and add the link to the json file in the code.
        if there are data pelase save the output as csv file called stockspred.csv.
        """,        
    )

    # Generate code using the GPTAssistantAgent
    #code_create.initiate_chat(model_client, message="Write a Python function that prints 'Hello, World!'")
    text_mention_termination = TextMentionTermination("TERMINATE")
    max_messages_termination = MaxMessageTermination(max_messages=25)
    termination = text_mention_termination | max_messages_termination
    team = MagenticOneGroupChat([code_create], 
                                 model_client=model_client,
                                 termination_condition=termination, max_turns=1)
    

    # Extract the generated code
    result = await Console(team.run_stream(task="Write a Python code to predict close price using sklearn for stocks.csv and print as table. ALso save as stockspred.csv"))
    last_message = result.messages
    code_blocks = extract_code(last_message[-1].content)

    shell_commands = "pip install yfinance matplotlib scikit-learn"
    python_code = ""
    if code_blocks:
        print("Code blocks extracted:", code_blocks)
        #shell_commands = next(block[1] for block in code_blocks if block[0] == 'sh' or block[0] == 'bash' or block[0] == 'shell')
        #print(shell_commands)
        python_code = next(block[1] for block in code_blocks if block[0] == 'python')

    print('Prediction Python Code only: ', python_code)

    local_executor = LocalCommandLineCodeExecutor(work_dir=work_dir)
    result = await local_executor.execute_code_blocks(
            code_blocks=[
                CodeBlock(language="python", code=python_code),
            ],
            cancellation_token=CancellationToken(),
        )
    #print(result)
    print('Prediction: ', result.output.strip())
```

- Now run the main function

```python
if __name__ == "__main__":
    asyncio.run(main())
```

- run the code and check the output
- There should be stocks.csv and stockspred.csv in the coding folder
- coding folder is where all the code and artifacts are stored.