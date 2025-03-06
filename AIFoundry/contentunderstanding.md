# Azure Content Understanding service - Process Image or Video

## Introduction

- Build the flow to create Schema -> Use Schema and send Image -> Wait for results -> Get results
- Using content understanding Azure AI services in Sweden Central (only this location worked)
- Our goal is to summarize video content and extract key information from images and videos.
- Also extract information based on provided Schema
- Sample schema is provided

## Prerequisites

- Azure Subscription
- Azure AI services
- Azure Content Understanding service
- Go to Keys and Endpoint for Content Understanding
- Using local Python environment
- Using REST API approach to python

## Steps

- Each step is created in code and extension of previous step
- First 2 steps are combined into one.

### Step 1: Create Schema

- import and load libraries

```
import requests
import json
import os
import time
from dotenv import load_dotenv
load_dotenv()
```

- Now write the code to create schema
- here is a sample schema

```
{
    "description": "Sample chart analyzer",
    "scenario": "image",
    "fieldSchema": {
      "fields": {
        "Title": {
          "type": "string",
          "kind" : "generate",
          "description": "Summarize the video"
        }
      }
    }
  }
```

- save the file as request_body.json
- Now load the environment variables and call the REST endpoint for schema creation
- i am using open source image for this example

```
# Define the variables
endpoint = os.getenv("CONTENT_UNDERSTANDING_ENDPOINT")  # Replace with your actual endpoint
analyzerId = "contentundersch"  # Replace with your actual analyzer ID
api_version = "2024-12-01-preview"
key = os.getenv("CONTENT_UNDERSTANDING_KEY")  # Replace with your actual subscription key
fileUrl = "https://cdn.pixabay.com/photo/2016/03/08/20/03/flag-1244649_1280.jpg"  # Replace with the URL of the file you want to analyze

url = f"{endpoint}/contentunderstanding/analyzers/{analyzerId}?api-version={api_version}"

headers = {
    "Ocp-Apim-Subscription-Key": key,
    "Content-Type": "application/json"
}

with open("request_body.json", "r") as file:
    data = file.read()

response = requests.put(url, headers=headers, data=data)

print(response.status_code)
print(response.text)
```

### Step 2: Use Schema and send Image

- now setup the image url
- create the URI with schema and send the URL in body

```
url = f"{endpoint}/contentunderstanding/analyzers/{analyzerId}:analyze?api-version={api_version}"

headers = {
    "Ocp-Apim-Subscription-Key": key,
    "Content-Type": "application/json"
}

data = json.dumps({"url": fileUrl})

response = requests.post(url, headers=headers, data=data)

print(response.status_code)
print(response.text)

result = json.loads(response.text)
operationId = result["id"]

url = f"{endpoint}/contentunderstanding/analyzers/{analyzerId}/results/{operationId}?api-version={api_version}"

headers = {
    "Ocp-Apim-Subscription-Key": key
}

response = requests.get(url, headers=headers)

print(response.status_code)
print(response.text)

result = json.loads(response.text)
operationId = result["id"]
```

- Note the operationId and use it in next step

### Step 3: Wait for results and display

- now create a new python file and use the operationId to get the results

```
import requests
import json
import os
import time
from dotenv import load_dotenv
load_dotenv()

# Define the variables
endpoint = os.getenv("CONTENT_UNDERSTANDING_ENDPOINT")  # Replace with your actual endpoint
analyzerId = "contentundersch"  # Replace with your actual analyzer ID
api_version = "2024-12-01-preview"
key = os.getenv("CONTENT_UNDERSTANDING_KEY")  # Replace with your actual subscription key
fileUrl = "https://cdn.pixabay.com/photo/2016/03/08/20/03/flag-1244649_1280.jpg"  # Replace with the URL of the file you want to analyze

operationId = "xxxxxxxxxxxxxxx"

url = f"{endpoint}/contentunderstanding/analyzers/{analyzerId}/results/{operationId}?api-version={api_version}"

headers = {
    "Ocp-Apim-Subscription-Key": key
}

response = requests.get(url, headers=headers)

print(response.status_code)
# print(response.text)

rs = json.loads(response.text)
#print(rs["status"])

rs1 = rs["result"]
#print(rs1)
# print(rs1["contents"])[0]["fields"]["Title"]["valueString"]
# Extract and print the valueString content
if rs1.get("contents"):
    for content in rs1["contents"]:
        title_field = content.get("fields", {}).get("Title", {})
        value_string = title_field.get("valueString")
        if value_string:
            print('Content extracted: ' , value_string)
```

- run the code and you will see the content extracted from the image
- Ouput will be like this

```
200
Content extracted:  Close-up of daisies and wildflowers in a garden
```

- More to come.