# Microsoft Autogen Code to Studio file conversion

## Introduction

- Create multiple agents
- Different personas
- Able to call tools with custom functions

## Prerequisites

- Azure Subscription
- Python 3.12 environment
- autogen 0.4.7
- autogenstudio - 0.4.1.1

## Steps

- Load imports

```
import asyncio
import os
from PyPDF2 import PdfReader
from autogen_agentchat.agents import AssistantAgent, UserProxyAgent
from autogen_agentchat.conditions import MaxMessageTermination, TextMentionTermination
from autogen_core.tools import Tool, FunctionTool
from autogen_core.code_executor import FunctionWithRequirements, with_requirements
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_ext.models.openai import AzureOpenAIChatCompletionClient
from autogen_agentchat.teams import RoundRobinGroupChat, MagenticOneGroupChat
from typing_extensions import Annotated
from autogen_ext.agents.file_surfer import FileSurfer
from textwrap import dedent, indent
from typing import Any, Callable, Generic, List, Sequence, Set, Tuple, TypeVar, Union
import json
import functools
from dotenv import load_dotenv
load_dotenv()

pdf_file_path = "DeepSeekR1-2501.12948v1.pdf"
```

- also loading environment variables
- specify the path to the pdf file
- Setup the Azure Open AI Chat Completion Client

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

- Create functions for tools calling by agents

```
@with_requirements(global_imports=["from PyPDF2 import PdfReader", "import os", "import json"], python_packages=["PyPDF2"])
def extract_text_from_pdf(filename: Annotated[str, "The full path to the PDF file"]) -> str:
    """
    Extracts all text from a PDF file.

    Args:
        pdf_path (str): Path to the PDF file.

    Returns:
        str: Extracted text from the PDF.
    """
    try:
        pdf_path = "DeepSeekR1-2501.12948v1.pdf"

        if filename != "":
            filename = pdf_path
        # Check if the file exists
        if not os.path.exists(filename):
            return f"Error: File '{filename}' does not exist."

        # Create a PdfReader object
        reader = PdfReader(filename)
        
        # Initialize an empty string to store the extracted text
        extracted_text = ""
        
        # Loop through all pages in the PDF and extract text
        for page in reader.pages:
            extracted_text += page.extract_text() + "\n"  # Add newline for separation between pages
        
        return extracted_text.strip()
    except Exception as e:
        return f"An error occurred: {e}"

# Step 2: Wrap the Function as a Tool Using FunctionTool
@with_requirements(global_imports=["os", "json"])
def save_files_to_json(directory: str) -> str:
    """
    Traverses all directories and files starting from the given directory
    and saves the list of file paths to a JSON file.

        Parameters:
        directory (str): The root directory to start traversal.
        output_file (str): The path to the output JSON file.
    """
    output_file = "file_list.json"
    file_list = []

    # # Traverse the directory tree
    # for dirpath, _, filenames in os.walk(directory):
    #     for filename in filenames:
    #         # Get the full file path
    #         full_path = os.path.join(dirpath, filename)
    #         file_list.append(full_path)

    # # Save the list of files to a JSON file
    # with open(output_file, 'w') as json_file:
    #     json.dump(file_list, json_file, indent=4)

    success = "success"
    # Traverse the directory tree
    for dirpath, _, filenames in os.walk(directory):
        for filename in filenames:
            # Get the full file path
            full_path = os.path.join(dirpath, filename)
            try:
                # Add file name and status to the list
                file_list.append({"file_name": filename, "status": success})
            except Exception as e:
                # If an error occurs, log it with a failed status
                file_list.append({'file_name': filename, 'status': f'failed - {str(e)}'})

    # Save the list of files with their statuses to a JSON file
    with open(output_file, 'w') as json_file:
        json.dump(file_list, json_file, indent=4)

    print(f"File list saved to {output_file}")

@with_requirements(global_imports=["os", "json"])
def parse_agent_response(response):
    """
    Parses the output of the agent's response and extracts structured information.

    Parameters:
        response (str): The raw response from the agent.

    Returns:
        dict: A dictionary containing parsed information.
    """
    try:
        # Attempt to parse as JSON if the response is structured
        parsed_response = json.loads(response)
        print("Parsed JSON Response:", parsed_response)
        return parsed_response
    except json.JSONDecodeError:
        # If not JSON, process as plain text
        print("Response is not in JSON format. Processing as plain text.")
        return {"text": response.strip()}
    
@with_requirements(global_imports=["os", "json"])
def parse_and_display_json(file_name: str) -> str:
    """
    Parses a JSON file and displays its contents in a readable format.

    Parameters:
    file_name (str): The name of the JSON file to parse.

    Returns:
    None
    """
    try:
        # Open and load the JSON file
        with open(file_name, 'r') as json_file:
            data = json.load(json_file)
        
        # Display the parsed data
        print("Parsed JSON Content:")
        for entry in data:
            print(f"File Name: {entry.get('file_name')}, Status: {entry.get('status')}")

    except FileNotFoundError:
        print(f"Error: The file '{file_name}' was not found.")
    except json.JSONDecodeError:
        print(f"Error: The file '{file_name}' is not a valid JSON file.")
    except Exception as e:
        print(f"An unexpected error occurred: {e}")
```

- now to the main function to setup agents
- Run the agents and tools
- Dump the output as JSON

```
# Step 4: Test the Agent with the Tool
if __name__ == "__main__":
    # Example usage of the tool directly
    pdf_file_path = "DeepSeekR1-2501.12948v1.pdf"  # Replace with your actual PDF file path
    
    #print("Testing the PDF Reader Tool:")
    #print(extract_text_from_pdf(pdf_file_path))

    # Example usage of the agent calling the tool
    print("\nTesting Agent with Tool:")
    # Step 2: Wrap the Function as a Tool Using FunctionTool
    pdf_reader_tool = FunctionTool(
        func=extract_text_from_pdf,
        description="Extracts text from a given PDF file. Provide the full file path as input.",
        name="pdf_reader"
    )

    # Step 3: Define an Agent and Add the Tool
    agent = AssistantAgent(
        name="assistant_with_pdf_reader",
        model_client=model_client,
        tools=[pdf_reader_tool],  # Add the tool to the agent's list of tools
        handoffs=["file_agent"],  # No handoffs for this agent
    )

    file_agent = AssistantAgent(
        name="file_agent",
        model_client=model_client,
        tools=[save_files_to_json],  # Add the tool to the agent's list of tools
        handoffs=["display_agent"],  # No handoffs for this agent
        system_message=f"""Please scan the current folder and create a list of all files available. Save this list of files to a JSON file named file_list.json. 
        If there are any errors encountered while processing the files, log those errors along with the file names in a separate JSON file named failed_files.json."""
    )

    display_agent = AssistantAgent(
        name="display_agent",
        model_client=model_client,
        tools=[parse_and_display_json],  # Add the tool to the agent's list of tools
        handoffs=["success_agent"],  # No handoffs for this agent
        system_message=f"""Please scan the current folder and diplay the contents of file_list.json."""
    )

    # Step 3: Define an Agent and Add the Tool
    success_agent = AssistantAgent(
        name="success_agent",
        model_client=model_client,
        tools=[],  # Add the tool to the agent's list of tools
        handoffs=[],  # No handoffs for this agent
        system_message=f"""Respond to user back saying all the file processing is completed. When complete respond TERMINATE"""
    )

    file_surfer = FileSurfer(
        name="file_surfer",
        model_client=model_client,
        description="Please scan the current folder ./data and preview the file contents.",
        #handoffs=["success_agent"],  # Handoff to the assistant agent after file selection
    )

    # Example usage of the agent calling the tool
    # print("\nTesting Agent with Tool:")
    
    # response = agent.chat(query)
    text_mention_termination = TextMentionTermination("TERMINATE")
    max_messages_termination = MaxMessageTermination(max_messages=25)
    termination = text_mention_termination | max_messages_termination
    # team = MagenticOneGroupChat([file_agent, file_surfer, success_agent], 
    #                             termination_condition=termination, 
    #                             max_turns=2,
    #                             model_client=model_client)
    team = RoundRobinGroupChat([file_agent, display_agent, success_agent], 
                                termination_condition=termination, 
                                max_turns=1)
    #query = f"Read and extract text from this file: DeepSeekR1-2501.12948v1.pdf"
    query = f"Can you loop files in current folder ./data and save the list of files in a JSON file?"
    # Extract the generated code
    #query = "Write a Python function that to chat last 6 months of Tesla stock price using yfinance library and print as table."
    #result = await Console(team.run_stream(task=query))
    response = asyncio.run(team.run(task=query))
    last_message = response.messages
    #print('Code created:', last_message[-1].content)

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
    print('Agent JSON:', team.dump_component().model_dump_json())
    print("-----------------------------------------------------------------------------------------")
```

- run the code and check the output

```
python autogenexp.py
```

- output

```json
{
   "provider":"autogen_agentchat.teams.RoundRobinGroupChat",
   "component_type":"team",
   "version":1,
   "component_version":1,
   "description":"A team that runs a group chat with participants taking turns in a round-robin fashion\nto publish a message to all.",
   "label":"RoundRobinGroupChat",
   "config":{
      "participants":[
         {
            "provider":"autogen_agentchat.agents.AssistantAgent",
            "component_type":"agent",
            "version":1,
            "component_version":1,
            "description":"An agent that provides assistance with tool use.",
            "label":"AssistantAgent",
            "config":{
               "name":"file_agent",
               "model_client":{
                  "provider":"autogen_ext.models.openai.AzureOpenAIChatCompletionClient",
                  "component_type":"model",
                  "version":1,
                  "component_version":1,
                  "description":"Chat completion client for Azure OpenAI hosted models.",
                  "label":"AzureOpenAIChatCompletionClient",
                  "config":{
                     "seed":42,
                     "temperature":0.0,
                     "model":"gpt-4o",
                     "api_key":"xxxxxxx",
                     "azure_endpoint":"https://xxxx.openai.azure.com",
                     "azure_deployment":"gpt-4o-2",
                     "api_version":"2024-10-21"
                  }
               },
               "tools":[
                  {
                     "provider":"autogen_core.tools.FunctionTool",
                     "component_type":"tool",
                     "version":1,
                     "component_version":1,
                     "description":"Create custom tools by wrapping standard Python functions.",
                     "label":"FunctionTool",
                     "config":{
                        "source_code":"def save_files_to_json(directory: str) -> str:\n    \"\"\"\n    Traverses all directories and files starting from the given directory\n    and saves the list of file paths to a JSON file.\n\n        Parameters:\n        directory (str): The root directory to start traversal.\n        output_file (str): The path to the output JSON file.\n    \"\"\"\n    output_file = \"file_list.json\"\n    file_list = []\n\n    success = \"success\"\n    # Traverse the directory tree\n    for dirpath, _, filenames in os.walk(directory):\n        for filename in filenames:\n            # Get the full file path\n            full_path = os.path.join(dirpath, filename)\n            try:\n                # Add file name and status to the list\n                file_list.append({\"file_name\": filename, \"status\": success})\n            except Exception as e:\n        # If an error occurs, log it with a failed status\n                file_list.append({'file_name': filename, 'status': f'failed - {str(e)}'})\n\n    # Save the list of files with their statuses to a JSON file\n    with open(output_file, 'w') as json_file:\n        json.dump(file_list, json_file, indent=4)\n\n    print(f\"File list saved to {output_file}\")\n",
                        "name":"save_files_to_json",
                        "description":"\nTraverses all directories and files starting from the given directory\nand saves the list of file paths to a JSON file.\n\n    Parameters:\n    directory (str): The root directory to start traversal.\n    output_file (str): The path to the output JSON file.\n",
                        "global_imports":[
                           
                        ],
                        "has_cancellation_support":false
                     }
                  }
               ],
               "handoffs":[
                  {
                     "target":"display_agent",
                     "description":"Handoff to display_agent.",
                     "name":"transfer_to_display_agent",
                     "message":"Transferred to display_agent, adopting the role of display_agent immediately."
                  }
               ],
               "model_context":{
                  "provider":"autogen_core.model_context.UnboundedChatCompletionContext",
                  "component_type":"chat_completion_context",
                  "version":1,
                  "component_version":1,
                  "description":"An unbounded chat completion context that keeps a view of the all the messages.",
                  "label":"UnboundedChatCompletionContext",
                  "config":{
                     
                  }
               },
               "description":"An agent that provides assistance with ability to use tools.",
               "system_message":"Please scan the current folder and create a list of all files available. Save this list of files to a JSON file named file_list.json. \n        If there are any errors encountered while processing the files, log those errors along with the file names in a separate JSON file named failed_files.json.",
               "model_client_stream":false,
               "reflect_on_tool_use":false,
               "tool_call_summary_format":"{result}"
            }
         },
         {
            "provider":"autogen_agentchat.agents.AssistantAgent",
            "component_type":"agent",
            "version":1,
            "component_version":1,
            "description":"An agent that provides assistance with tool use.",
            "label":"AssistantAgent",
            "config":{
               "name":"display_agent",
               "model_client":{
                  "provider":"autogen_ext.models.openai.AzureOpenAIChatCompletionClient",
                  "component_type":"model",
                  "version":1,
                  "component_version":1,
                  "description":"Chat completion client for Azure OpenAI hosted models.",
                  "label":"AzureOpenAIChatCompletionClient",
                  "config":{
                     "seed":42,
                     "temperature":0.0,
                     "model":"gpt-4o",
                     "api_key":"xxxxxxx",
                     "azure_endpoint":"https://xxxxx.openai.azure.com",
                     "azure_deployment":"gpt-4o-2",
                     "api_version":"2024-10-21"
                  }
               },
               "tools":[
                  {
                     "provider":"autogen_core.tools.FunctionTool",
                     "component_type":"tool",
                     "version":1,
                     "component_version":1,
                     "description":"Create custom tools by wrapping standard Python functions.",
                     "label":"FunctionTool",
                     "config":{
                        "source_code":"def parse_and_display_json(file_name: str) -> str:\n    \"\"\"\n    Parses a JSON file and displays its contents in a readable format.\n\n    Parameters:\n    file_name (str): The name of the JSON file to parse.\n\n    Returns:\n    None\n    \"\"\"\n    try:\n        # Open and load the JSON file\n        with open(file_name, 'r') as json_file:\n            data = json.load(json_file)\n\n        # Display the parsed data\n        print(\"Parsed JSON Content:\")\n        for entry in data:\n            print(f\"File Name: {entry.get('file_name')}, Status: {entry.get('status')}\")\n\n    except FileNotFoundError:\n        print(f\"Error: The file '{file_name}' was not found.\")\n    except json.JSONDecodeError:\n        print(f\"Error: The file '{file_name}' is not a valid JSON file.\")\n    except Exception as e:\n        print(f\"An unexpected error occurred: {e}\")\n",
                        "name":"parse_and_display_json",
                        "description":"\nParses a JSON file and displays its contents in a readable format.\n\nParameters:\nfile_name (str): The name of the JSON file to parse.\n\nReturns:\nNone\n",
                        "global_imports":[
                           
                        ],
                        "has_cancellation_support":false
                     }
                  }
               ],
               "handoffs":[
                  {
                     "target":"success_agent",
                     "description":"Handoff to success_agent.",
                     "name":"transfer_to_success_agent",
                     "message":"Transferred to success_agent, adopting the role of success_agent immediately."
                  }
               ],
               "model_context":{
                  "provider":"autogen_core.model_context.UnboundedChatCompletionContext",
                  "component_type":"chat_completion_context",
                  "version":1,
                  "component_version":1,
                  "description":"An unbounded chat completion context that keeps a view of the all the messages.",
                  "label":"UnboundedChatCompletionContext",
                  "config":{
                     
                  }
               },
               "description":"An agent that provides assistance with ability to use tools.",
               "system_message":"Please scan the current folder and diplay the contents of file_list.json.",
               "model_client_stream":false,
               "reflect_on_tool_use":false,
               "tool_call_summary_format":"{result}"
            }
         },
         {
            "provider":"autogen_agentchat.agents.AssistantAgent",
            "component_type":"agent",
            "version":1,
            "component_version":1,
            "description":"An agent that provides assistance with tool use.",
            "label":"AssistantAgent",
            "config":{
               "name":"success_agent",
               "model_client":{
                  "provider":"autogen_ext.models.openai.AzureOpenAIChatCompletionClient",
                  "component_type":"model",
                  "version":1,
                  "component_version":1,
                  "description":"Chat completion client for Azure OpenAI hosted models.",
                  "label":"AzureOpenAIChatCompletionClient",
                  "config":{
                     "seed":42,
                     "temperature":0.0,
                     "model":"gpt-4o",
                     "api_key":"xxxxxxx",
                     "azure_endpoint":"https://xxxxxxx.openai.azure.com",
                     "azure_deployment":"gpt-4o-2",
                     "api_version":"2024-10-21"
                  }
               },
               "tools":[
                  
               ],
               "handoffs":[
                  
               ],
               "model_context":{
                  "provider":"autogen_core.model_context.UnboundedChatCompletionContext",
                  "component_type":"chat_completion_context",
                  "version":1,
                  "component_version":1,
                  "description":"An unbounded chat completion context that keeps a view of the all the messages.",
                  "label":"UnboundedChatCompletionContext",
                  "config":{
                     
                  }
               },
               "description":"An agent that provides assistance with ability to use tools.",
               "system_message":"Respond to user back saying all the file processing is completed. When complete respond TERMINATE",
               "model_client_stream":false,
               "reflect_on_tool_use":false,
               "tool_call_summary_format":"{result}"
            }
         }
      ],
      "termination_condition":{
         "provider":"autogen_agentchat.base.OrTerminationCondition",
         "component_type":"termination",
         "version":1,
         "component_version":1,
         "label":"OrTerminationCondition",
         "config":{
            "conditions":[
               {
                  "provider":"autogen_agentchat.conditions.TextMentionTermination",
                  "component_type":"termination",
                  "version":1,
                  "component_version":1,
                  "description":"Terminate the conversation if a specific text is mentioned.",
                  "label":"TextMentionTermination",
                  "config":{
                     "text":"TERMINATE"
                  }
               },
               {
                  "provider":"autogen_agentchat.conditions.MaxMessageTermination",
                  "component_type":"termination",
                  "version":1,
                  "component_version":1,
                  "description":"Terminate the conversation after a maximum number of messages have been exchanged.",
                  "label":"MaxMessageTermination",
                  "config":{
                     "max_messages":25,
                     "include_agent_event":false
                  }
               }
            ]
         }
      },
      "max_turns":1
   }
}
```

- more to come
- MagenticOneGroupChat doesn't work as expected for dumping as JSON