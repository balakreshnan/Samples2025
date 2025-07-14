# Computer use to run local applications

## Introduction

- To show how to use computer-use-preview open ai model to run local applications.
- This is a preview of the model, and it may not be fully functional or may have limitations.
- This model is designed to assist with running local applications by providing guidance and instructions.
- Check the regional availability of the model in your area.
- This is just to show case the capabilities of the model, and it may not be suitable for all use cases.
- Only for demonstration purposes.

## Pre-requisites

- Azure Subscription
- Azure AI Foundry project
- Computer-use-preview model deployed in your Azure AI Foundry project
- Openai Python SDK installed
- Using Visual Studio Code or any other IDE that supports Python development
- Python version 3.12

## Steps to run local applications

- Import libraries

```
import argparse
import asyncio
import logging
import os

import cua
import local_computer
import openai
```

- Load Environment Variables

```
from dotenv import load_dotenv

# Load environment variables
load_dotenv()
```

- Function to call the computer-use-preview model
- idea here is to open work and just type hello world in the document

```
async def main():

    # https://github.com/Azure-Samples/computer-use-model/tree/main/computer-use

    logging.basicConfig(level=logging.WARNING, format="%(message)s")
    logger = logging.getLogger(__name__)
    logger.setLevel(logging.DEBUG)

    parser = argparse.ArgumentParser()
    parser.add_argument("--instructions", dest="instructions",
        default="Open Word and type Hello world.",
        help="Instructions to follow")
    parser.add_argument("--model", dest="model",
        default="computer-use-preview")
    parser.add_argument("--endpoint", default="azure",
        help="The endpoint to use, either OpenAI or Azure OpenAI")
    parser.add_argument("--autoplay", dest="autoplay", action="store_true",
        default=True, help="Autoplay actions without confirmation")
    parser.add_argument("--environment", dest="environment", default="linux")
    args = parser.parse_args()

    if args.endpoint == "azure":
        client = openai.AsyncAzureOpenAI(
            azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
            api_key=os.environ["AZURE_OPENAI_KEY"],
            api_version="2025-03-01-preview",
        )
    else:
        client = openai.AsyncOpenAI()

    model = args.model

    # Computer is used to take screenshots and send keystrokes or mouse clicks
    computer = local_computer.LocalComputer()
    print(f"Computer environment: {computer.environment}")

    # Scaler is used to resize the screen to a smaller size
    computer = cua.Scaler(computer, (1024, 768))

    # Agent to run the CUA model and keep track of state
    agent = cua.Agent(client, model, computer)

    # Get the user request
    if args.instructions:
        user_input = args.instructions
    else:
        user_input = input("Please enter the initial task: ")

    logger.info(f"User: {user_input}")
    agent.start_task()
    while True:
        if not user_input and agent.requires_user_input:
            logger.info("")
            user_input = input("User: ")
        if user_input.lower() in ["exit", "quit", "stop"]:
            logger.info("Exiting...")
            break
        await agent.continue_task(user_input)
        user_input = ""
        if agent.requires_consent and not args.autoplay:
            input("Press Enter to run computer tool...")
        elif agent.pending_safety_checks and not args.autoplay:
            logger.info(f"Safety checks: {agent.pending_safety_checks}")
            input("Press Enter to acknowledge and continue...")
        if agent.reasoning_summary:
            logger.info("")
            logger.info(f"Action: {agent.reasoning_summary}")
        for action, action_args in agent.actions:
            logger.info(f"  {action} {action_args}")
        if agent.messages:
            logger.info("")
            logger.info(f"Agent: {"".join(agent.messages)}")
```

- now create main function to run the above code

```
if __name__ == "__main__":
    asyncio.run(main())
```

- now save the file as `local_computer.py` in your project directory.
- Run the script using the command line or your IDE.
- Observe the output and interactions in the console.
- You should see the computer opening Word and typing "Hello world" in the document.