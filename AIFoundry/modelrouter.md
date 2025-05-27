# Azure AI foundry Model Router

## Introduction

- Using Azure AI Foundry Model Router, you can route requests to different models based on the content of the request.
- To show how model router works, we will use a simple example of routing requests to different models based on the content of the request.
- We are using simple streamlit app to demonstrate the model router.
- Model router has to be deployed in Azure AI Foundry.

## Prerequisites

- Azure Subscription
- Azure AI Foundry
- Deploy model router in Azure AI Foundry from Model Catalog
- I am using sweden central as the region for the deployment.
- Streamlit installed in your local environment
- using Visual Studio Code as the IDE
- Python 3.13 or later installed in your local environment
- Install or update openai sdk
- install and update streamlit sdk

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/modelrouter-1.jpg 'RagChat')

- Supported model as of writing the article:

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/modelrouter-2.jpg 'RagChat')

## Code

- First import the required libraries
- Load the environment variables from the .env file

```
import asyncio
from datetime import datetime
import streamlit as st
import os
import openai
from typing import List, Sequence

from dotenv import load_dotenv
load_dotenv()
```

- now configure the openai client with the model router endpoint and the api key

```
model_name = os.getenv("AZURE_OPENAI_DEPLOYMENT")
endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
model_name = "model-router"
deployment = "model-router"

client = openai.AzureOpenAI(
        azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"), 
        api_key = os.getenv("AZURE_OPENAI_API_KEY"),
        api_version = "2024-12-01-preview",
        azure_deployment=model_name,
    )
```

- now create a function to call the model router

```
def model_router(query: str) -> str:

    txtreturn = ""
    txtmodelused = ""
    starttime = datetime.now()
    txtmessages=[
        {
            "role": "system",
            "content": "You are a helpful assistant.",
        },
        {
            "role": "user",
            "content": f"{query}",
        }
    ]

    response = client.chat.completions.create(
        model=model_name,
        messages=txtmessages,
        max_tokens=8192,
        temperature=0.7,
        top_p=0.95,
        frequency_penalty=0.0,
        presence_penalty=0.0,
    )
    endtime = datetime.now()
    
    txtreturn = response.choices[0].message.content.strip() + "\n"
    txtreturn += f"Response Time: {endtime - starttime}\n"
    # txtreturn += "Response Model Used: " + response.model + "\n"

    txtmodelused += "Response Model Used: " + response.model + "\n"

    return txtreturn, txtmodelused
```

- i am sending back model selected by the model router and the response time in the response.
- Both are responsed as separate strings.
- Now lets create the actual streamlit app to take input from the user and display the response from the model router.

```
def routermain():
    st.title("Model Router")
    st.write("This application routes queries to different models based on the content of the query.")
    
    query = st.text_input("Enter your query:")
    
    if st.button("Submit"):
        if query:
            with st.spinner("Processing..."):
                txtreturn, txtmodelused = model_router(query)
                st.success("Response received!")
                st.write(txtmodelused)
                st.text_area("Response", value=txtreturn, height=300)
        else:
            st.error("Please enter a query before submitting.")
```

- now we can run the streamlit app

```if __name__ == "__main__":
    routermain()
```

- Save the code in a file named `modelrouter.py`.
- Run the streamlit app using the following command:

```
streamlit run modelrouter.py
```

- Check the various outputs by entering different queries.
- when you use describe, it will route the request to reasoning model.
- Simplpe queries will be routed to the gpt 4o nano or mini models.

- here are some samples i am testing

```
can you detail write about how science can use large lanugage model to improve scientific discovery
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/modelrouter-3.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/modelrouter-4.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/modelrouter-5.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/modelrouter-6.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/modelrouter-7.jpg 'RagChat')

- done for now
- Keep going, keep learning, keep sharing
- feel free to reach out to and provide feedback or suggestions or thoughts