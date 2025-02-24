# Microsoft Autogen 0.4 - Code Converter - Java to Python

## Introduciton

- Idea here is to show how to use autogen 0.4 and above to migrate code
- From Source Code directory to Destination Source Code directory
- From Java to Python
- This is a simple example to show how to use autogen
- This is not a production ready code
- Ability to serialize and deserialize objects
- We are using MagenticOneGroupChat for team.

## Pre-requisites

- Install autogen 0.4 and above
- Azure Subscription
- Python 3.12 and above
- install MagenticOne

## Steps

- Create a new directory called scode to store source java code files like .java files
- Create another new directory called dcode to store destination python code files like .py files
- Now import all the libraries

```
import asyncio
import os
import re
from PyPDF2 import PdfReader
from openai import AzureOpenAI
from autogen_agentchat.agents import AssistantAgent, UserProxyAgent
from autogen_agentchat.conditions import MaxMessageTermination, TextMentionTermination
from autogen_core.tools import Tool, FunctionTool
from autogen_core.code_executor import FunctionWithRequirements, with_requirements
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_ext.models.openai import AzureOpenAIChatCompletionClient
from autogen_agentchat.teams import RoundRobinGroupChat, MagenticOneGroupChat
from autogen_ext.agents.magentic_one import MagenticOneCoderAgent
from typing_extensions import Annotated
from autogen_ext.agents.file_surfer import FileSurfer
from textwrap import dedent, indent
from autogen_ext.teams.magentic_one import MagenticOne
from autogen_agentchat.ui import Console
from typing import Any, Callable, Generic, List, Sequence, Set, Tuple, TypeVar, Union
import json
import functools
from autogen_core.code_executor import ImportFromModule
from dotenv import load_dotenv
load_dotenv()
```

- I am loading all the environment variables for Azure open AI
- Now configure variables

```
source_file_path = "./scode/"
```

- Configure Model Client

```
model_client = AzureOpenAIChatCompletionClient(model="gpt-4o",
                                               azure_deployment="gpt-4o-2", 
                                               azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"), 
                                               api_key=os.getenv("AZURE_OPENAI_API_KEY"), 
                                               api_version="2024-10-21",
                                               temperature=0.0,
                                               seed=42,
                                               maz_tokens=4096)
```

- Above is used by the MagenticOneGroupChat
- Now also create AzureOpenAI object

```
client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"), 
  api_key=os.getenv("AZURE_OPENAI_API_KEY"),  
  api_version="2024-10-21",
)

model_name = os.getenv("AZURE_OPENAI_DEPLOYMENT")
```

- Now write the functions to read folders and files

```
def read_code_in_folder(folder_path: str) -> str:
    java_files = []
    for root, dirs, files in os.walk(folder_path):
        for file in files:
            if file.endswith('.java'):
                java_files.append(os.path.join(root, file))

    write_to_file(java_files, 'filetoprocess.txt')
    return java_files

def write_to_file(file_list, output_filename):
    with open(output_filename, 'w+') as f:
        for file_path in file_list:
            f.write(f"{file_path}\n")
```

- create a list of files to process
- Now write the function to convert java code to python and save in destination folder

```
def convertcodellm(contents: str) -> str:
    returntxt = ""

    rfttext = ""


    message_text = [
    {"role":"system", "content":f"""You are AI Codine Expert. Convert Java code to Python code using LLM model. 
     Here is the code to convert Java code to Python code using LLM model:
     {contents}

    ."""}, 
    {"role": "user", "content": f""" Based on the code provided please convert to python code."""}]

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

def extract_python_code(text):  
    # Use regular expressions to find the Python code block  
    match = re.search(r'```python(.*?)```', text, re.DOTALL)  
    if match:  
        # Extract and clean the code  
        code = match.group(1).strip()  
        # Fix any formatting issues, like missing newlines  
        code = code.replace('def home():', 'def home():\n').replace("')", "')\n")  
        return code  
    return None 

@with_requirements(global_imports=["from PyPDF2 import PdfReader", "import os", "import json"], python_packages=["PyPDF2"])
def convert_code_in_folder(folder_path: str) -> str:
    java_files = []
    java_contents = ""
    for root, dirs, files in os.walk(folder_path):
        for file in files:
            if file.endswith('.java'):
                file_path = os.path.join(root, file)
                java_files.append(file_path)
                # Read the content of each Java file
                with open(file_path, 'r', encoding='utf-8') as f:
                    java_contents += f"// File: {file_path}\n"
                    java_contents += f.read()
                    java_contents += "\n\n"  # Add some separation between files

                #now convert the java code to python code
                python_code = convertcodellm(java_contents)
                #write the python code to file
                with open('./dcode/' + file.replace(".java","") + '.py', 'w+') as f:
                    f.write(extract_python_code(python_code))

    return java_contents
```

- Now additional file is for printing metrics on usage

```
def printmetrics(response):
    print("-----------------------------------------------------------------------------------------")
    print("Now it's time to print the token usage metrics for the agents.")
    print("-----------------------------------------------------------------------------------------")
    # Initialize a dictionary to store per-agent token usage
    agent_token_usage = {}
    total_prompt_tokens = 0
    total_completion_tokens = 0

    # Process each message
    for message in response.messages:
        source = message.source
        usage = message.models_usage

        if usage:
            prompt_tokens = usage.prompt_tokens
            completion_tokens = usage.completion_tokens

            # Update per-agent token usage
            if source != "file_agent":            
                if source not in agent_token_usage:
                    agent_token_usage[source] = {"prompt_tokens": 0, "completion_tokens": 0}

                agent_token_usage[source]["prompt_tokens"] += prompt_tokens
                agent_token_usage[source]["completion_tokens"] += completion_tokens

                # Update total token counts
                total_prompt_tokens += prompt_tokens
                total_completion_tokens += completion_tokens
            else:
                print(f"File Agent: Message = {message}")

    # Display per-agent token usage
    for agent, usage in agent_token_usage.items():
        print(f"{agent}: Prompt Tokens = {usage['prompt_tokens']}, Completion Tokens = {usage['completion_tokens']}")

    # Display total token usage
    print(f"Total Prompt Tokens: {total_prompt_tokens}")
    print(f"Total Completion Tokens: {total_completion_tokens}")
    print(f"Total Token Usage: {total_prompt_tokens + total_completion_tokens}")
    print("-----------------------------------------------------------------------------------------")
```

- Now write the main function to call all the functions
- Now create buildagent function and then call from Main

```
def build_agent():
    # Example usage of the tool directly
    code_reader_tool = FunctionTool(
        func=read_code_in_folder,
        description="Loop through the folder and take .java file and create a list of java files to process. Provide the full directory path as input.",
        name="file_reader",
        global_imports=["os", "json", ImportFromModule("PyPDF2", ("PdfReader",)),]
    )

    # Step 2: Define an Agent and Add the Tool
    agent = AssistantAgent(
        name="Java_file_reader",
        model_client=model_client,
        tools=[code_reader_tool],  # Add the tool to the agent's list of tools
        handoffs=["convert_agent"],  # No handoffs for this agent
        description="Read the content of Java files in the folder and create a list of files to process."
    )

    convert_reader_tool = FunctionTool(
        func=convert_code_in_folder,
        description="Read the content and convert java code to python code using model memory and save to ./dcode folder.",
        name="convert_reader",
        global_imports=["os", "json", ImportFromModule("PyPDF2", ("PdfReader",)),]
    )

    # Step 3: Define an Agent and Add the Tool
    convert_agent = AssistantAgent(
        name="convert_agent",
        model_client=model_client,
        tools=[convert_reader_tool],  # Add the tool to the agent's list of tools
        #handoffs=["code_write_agent"],  # No handoffs for this agent
        description="Convert Java code to Python code using model memory and save to ./dcode folder using tool provided."
    )

    text_mention_termination = TextMentionTermination("TERMINATE")
    max_messages_termination = MaxMessageTermination(max_messages=25)
    termination = text_mention_termination | max_messages_termination

    team = MagenticOneGroupChat([agent, convert_agent], 
                                termination_condition=termination, 
                                max_turns=1,
                                model_client=model_client)

    query = f"Can you loop files in current folder ./scode and convert to python and store to ./dcode folder?"
    # summarize ./papers
    # Extract the generated code
    response = asyncio.run(team.run(task=query))
    #response = asyncio.run(team.run_stream(task=query))
    # last_message = response.messages
    print("-----------------------------------------------------------------------------------------")

    for message in response.messages:
        print("-----------------------------------------------------------------------------------------")
        print('Messages: ' , message)
        print('Messages Source: ' , message.source)
        print('Messages Content: ' , message.content)
        print("-----------------------------------------------------------------------------------------")

    # printmetrics(response)

    #print('Response: \n' , response)
    #parse_agent_response(response)
    #print("-----------------------------------------------------------------------------------------")
    print('Agent JSON:', team.dump_component().model_dump_json())
    #serialized = team.dump_component().model_dump(mode='python')
    #print(f"Serialized team: {json.dumps(serialized)}")
    #print("-----------------------------------------------------------------------------------------")
    #print(convert_reader_tool.dump_component().model_dump_json())
    #print("-----------------------------------------------------------------------------------------")
    

# Step 4: Test the Agent with the Tool
if __name__ == "__main__":
    #asyncio.run(build_agent())
    build_agent()
```

- Save and run the code

```
python codeconv.py
```

- Now you can see the java code files are converted to python code files in destination folder
- Also the entire code will be dumped as JSON file that we can use to create in autogen studio.
- Entire code will be serialized and printed as JSON String.