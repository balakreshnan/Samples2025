# Microsoft Agent Framework - Sending local Image file

## Introduction

- Build a engineering drawing agent using Microsoft Agent Framework.
- idea here is to load autocad drawing files and convert them to images.
- Use Microsoft Agent Framework to create an agent that can process these images.
- Using agent with gpt 5 chat model to understand the drawing and answer questions.
- Make it available as a web app using streamlit.
- Use local image files to test the agent.
- Using responses client from open ai to use in agent framework.

## Prerequisites

- Azure Subscription
- Azure AI Foundry
- Deploy gpt 5 chat model
- Streamlit
- python 3.13 or above
- Install required python packages
    - pip install msagentframework
    - pip install streamlit
    - pip install openai
    - pip install python-dotenv
- Create a .env file with openai api key
    - AZURE_OPENAI_RESPONSES_DEPLOYMENT_NAME="gpt-5-chat"
    - AZURE_OPENAI_ENDPOINT="https://your-endpoint.openai.azure.com/"
    - AZURE_OPENAI_KEY="your-api-key"


## Code

- Create a python file app.py and add the following code

```python
# Copyright (c) Microsoft. All rights reserved.

import asyncio
import base64
from pathlib import Path

from agent_framework import ChatMessage, DataContent, Role, TextContent, UriContent
from agent_framework.azure import AzureOpenAIResponsesClient
from azure.identity import AzureCliCredential
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

"""
Azure OpenAI Responses Client with Image Analysis Example

This sample demonstrates using Azure OpenAI Responses for image analysis and vision tasks,
showing multi-modal messages combining text and image content.
"""


async def main():
    print("=== Azure Responses Agent with Image Analysis ===")

    # 1. Create an Azure Responses agent with vision capabilities
    agent = AzureOpenAIResponsesClient(credential=AzureCliCredential()).create_agent(
        name="VisionAgent",
        instructions="You are a helpful agent that can analyze images.",
    )

    image_path = "img/csifactoryengdraw1.jpg"
    
    
    image_path = Path("img/csifactoryengdraw1.jpg")
    
    with image_path.open("rb") as f:
        encoded_bytes = base64.b64encode(f.read())
    
    base64_string = encoded_bytes.decode("utf-8")
    # print(base64_string)
    # image_uri = f"data:image/jpg;base64,{base64_string}"
    data_uri = f"data:image/jpeg;base64,{base64_string}"

    # 2. Create a simple message with both text and image content
    # user_message = ChatMessage(
    #     role="user",
    #     contents=[
    #         TextContent(text="What do you see in this image?"),
    #         UriContent(
    #             uri="https://upload.wikimedia.org/wikipedia/commons/thumb/d/dd/Gfp-wisconsin-madison-the-nature-boardwalk.jpg/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg",
    #             media_type="image/jpeg",
    #         ),
    #     ],
    # )
    user_message = ChatMessage(
        role=Role.USER,
        contents=[TextContent(text="What's in this image?"), 
                  DataContent(
                    uri=data_uri, media_type="image/jpeg"
                    )
                ],
    )

    # 3. Get the agent's response
    print("User: What do you see in this image? [Image provided]")
    result = await agent.run(user_message)
    print(f"Agent: {result.text}")
    print(f"Usage Details: {result.usage_details}")
    print()
    


if __name__ == "__main__":
    asyncio.run(main())
```

- Create a folder named img and add an image file named csifactoryengdraw1.jpg
- if you have a different image file, update the code with the correct file name.
- Run the app using the command
    - streamlit run app.py
    - This will open a new browser window with the streamlit app.
    - You should see the agent's response to the image.
    - Try with different questions and validate the answers.
- This is to show multimodal using images with microsoft agent framework.
- Have fun building agents.

![info](https://github.com/balakreshnan/Samples2025/blob/main/msagentframework/images/imgagent-1.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/msagentframework/images/imgagent-2.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/msagentframework/images/imgagent-3.jpg 'RagChat')

## Conclusion

- Microsoft Agent Framework makes it easy to build agents that can process images.
- You can use local image files to test the agent.
- You can use different models from Azure OpenAI.
- You can build different types of agents using this framework.
- Integrate with different applications using streamlit or other web frameworks.
- Idea here is to show case building new applications.