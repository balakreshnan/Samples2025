# Microsoft Autogen 0.4 - Read files from PDF and summarize using Azure OpenAI

## Introduciton

- Idea here is to show how to use autogen 0.4 and summarize a lots of pdf files using Azure OpenAI
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

- Loop all the PDF files in a folder and summarize each PDF file
- Agent will summarize each PDF file
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
- Read PDF contents from a folder

```
def read_pdfs_in_folder(folder_path: str) -> str:
    pdf_contents = {}

    for root, dirs, files in os.walk(folder_path):
        for file in files:
            if file.lower().endswith('.pdf'):
                file_path = os.path.join(root, file)
                try:
                    reader = PdfReader(file_path)
                    text = ""
                    for page in reader.pages:
                        text += page.extract_text()
                    pdf_contents[file] = text
                except Exception as e:
                    print(f"Error reading {file}: {str(e)}")

    return pdf_contents
```

- Now write the main function to call all the functions
- Now create Main function

```
if __name__ == "__main__":
    # Example usage of the tool directly
    pdf_reader_tool = FunctionTool(
        func=read_pdfs_in_folder,
        description="Extracts text from a given PDF files in the folder. Provide the full directory path as input.",
        name="pdf_reader",
        global_imports=["os", "json", ImportFromModule("PyPDF2", ("PdfReader",)),]
    )

    # Step 3: Define an Agent and Add the Tool
    agent = AssistantAgent(
        name="assistant_with_pdf_reader",
        model_client=model_client,
        tools=[pdf_reader_tool],  # Add the tool to the agent's list of tools
        handoffs=["file_agent"],  # No handoffs for this agent
    )

    text_mention_termination = TextMentionTermination("TERMINATE")
    max_messages_termination = MaxMessageTermination(max_messages=25)
    termination = text_mention_termination | max_messages_termination

    team = MagenticOneGroupChat([agent], 
                                termination_condition=termination, 
                                max_turns=1,
                                model_client=model_client)

    query = f"Can you loop files in current folder ./papers and summarize each pdf?"
    # summarize ./papers
    # Extract the generated code
    response = asyncio.run(team.run(task=query))
    last_message = response.messages
    print("-----------------------------------------------------------------------------------------")

    for message in response.messages:
        print("-----------------------------------------------------------------------------------------")
        print('Messages: ' , message)
        print('Messages Source: ' , message.source)
        print('Messages Content: ' , message.content)
        print("-----------------------------------------------------------------------------------------")

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

    #print('Response: \n' , response)
    #parse_agent_response(response)
    print("-----------------------------------------------------------------------------------------")
    #print('Agent JSON:', team.dump_component().model_dump_json())
    serialized = team.dump_component().model_dump(mode='python')
    print(f"Serialized team: {json.dumps(serialized)}")
    print("-----------------------------------------------------------------------------------------")
```

- Save and run the code

```
python codeconv.py
```

- Now the output should be summarized by each PDF and shown in the screen.
- Also the entire code will be dumped as JSON file that we can use to create in autogen studio.
- Entire code will be serialized and printed as JSON String.