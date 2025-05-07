# Azure AI Foundry - Phi4 Multimodal Inference

## Overview

- To show how to send image and text to phi 4 multimodal inference endpoint using Azure AI Foundry.
- The code will send a request to the endpoint and print the response.

## Prerequisites

- Azure subscription
- Azure AI Foundry workspace with a deployed Phi 4 multimodal inference model
- Deploy Phi 4 Multimodal model deployment in Azure AI Foundry
- Get the endpoint URL and API key from the Azure AI Foundry workspace

## Code

- First install the required libraries:

```bash
pip install azure.ai.inference azure.identity requests
```

- import the required libraries:

```python
import os
from azure.ai.inference import ChatCompletionsClient
from azure.core.credentials import AzureKeyCredential
from azure.ai.inference.models import UserMessage, TextContentItem, ImageContentItem
import base64
from PIL import Image
import io

from dotenv import load_dotenv
load_dotenv()
```

- Now Setup the client

```
client = ChatCompletionsClient(
    endpoint="https://foundryprojectname.services.ai.azure.com/models",
    credential=AzureKeyCredential(os.environ["AZURE_AI_INFERENCING_KEY"]),
)
```

- Now setup the main function to send the request to the endpoint:

```python
def main():
    from azure.ai.inference.models import SystemMessage, UserMessage

    # Read and encode the local image
    image_path = "./images/agenticaiimg-1.jpeg"  # Replace with your local image path
    with Image.open(image_path) as img:
        # Convert image to bytes
        buffered = io.BytesIO()
        img.save(buffered, format=img.format if img.format else "JPEG")
        image_bytes = buffered.getvalue()
        # Encode to base64
        base64_image = base64.b64encode(image_bytes).decode("utf-8")

    # Prepare the multimodal input with base64-encoded image
    messages = [
        UserMessage(
            content=[
                ImageContentItem(
                    image_url={
                        "url": f"data:image/jpeg;base64,{base64_image}"
                    }
                ),
                TextContentItem(text="Describe this image in detail.")
            ]
        )
    ]

    # Make the chat completions request
    response = client.complete(
        model="Phi-4-multimodal-instruct",
        messages=messages,
        max_tokens=500,
        temperature=0.0
    )

    # Print the response
    print("Here is output: ", response.choices[0].message.content)
```

- Finally, run the main function:

```python
if __name__ == "__main__":
    main()
```

- Run the code and check the output.

```
Here is output:  In a futuristic cityscape, a diverse group of people, including a baby in a stroller, gather around a holographic projection of a human head, which is surrounded by a network of blue digital connections. The scene is set against a backdrop of towering skyscrapers and a clear sky, with a police car labeled "POLICE" and "ZHIVIBUKA" in the foreground, and a billboard displaying a profile of a person with a play button icon.
```

- Done! You have successfully sent a request to the Phi 4 multimodal inference endpoint and received a response. You can modify the input image and text to test different scenarios.

## conclusion

- Now try with different images and text to see how the model responds.