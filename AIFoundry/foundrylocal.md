# Azure AI Foundry Running models locally

## Introduction

- Azure Subscription
- Azure AI Foundry
- Install Azure AI Foundry locally Server - https://github.com/ai-platform-microsoft/WindowsAIFoundry/releases/tag/0.2.9245

## Steps

- First download the model

```
foundry --version
```

- Now download the model using the following command

```
foundry model run deepseek-r1-1.5b-cuda
```

- to see the model id

```
foundry cache list
```

- Here is the code

```
# app.py
from azure.ai.inference import ChatCompletionsClient
from azure.core.credentials import AzureKeyCredential

# create a chat completions client
client = ChatCompletionsClient(endpoint="http://127.0.0.1:5272/v1", credential=AzureKeyCredential("notneeded"))

# run a chat completion using the inferencing client
response = client.complete(
    model="deepseek-r1-distill-qwen-1.5b-cuda-int4-rtn-block-32",
    messages=[
        {"role": "system", "content": "You are a helpful writing assistant"},
        {"role": "user", "content": "can you detail steps on creating quantum computer?"},
    ],
    stream=True,
    max_tokens=4000,
)

# print out the streaming response
for item in response:
    print(item.choices[0].delta.content, end="")
```

- now run the code

```
python app.py
```

- done