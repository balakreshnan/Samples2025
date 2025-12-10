# Deploy Yolo 11n in Azure Machine Learning Managed Online Endpoint

## Introduction

- Ability to deploy Yolo 11n model in Azure Machine Learning Managed Online Endpoint.
- Download Yolo 11n model from Hugging Face Model Hub.
- Create a scoring script to load the model and perform inference.
- Test and validate the model in compute instance
- Register the model in Azure Machine Learning workspace.
- Deploy the model to a managed online endpoint.
- Test the deployed endpoint with sample input data.
- Create Environment with required dependencies.

## Create Azure Machine Learning Workspace

- Create a brand new Azure Machine Learning Workspace in Azure Portal.
- Create a new resource group or use an existing one.
- Now for storage account provide these RBAC roles to the AML workspace managed identity.
  - Storage Blob Data Contributor
  - Storage File Privileged Data Contributor
- Create a new Azure Machine Learning Compute Instance in the AML workspace.
- Choose the appropriate size for the compute instance based on your requirements.
- SKU i am using is Standard_D32a_v4

## Code Example

### Setup Compute Instance

- First create a new compute instance in Azure Machine Learning workspace.
- Now go to terminal and run the following commands to create a yolo11env python virtual environment with required dependencies.

```bash
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r

conda create --name yolo11env -y python=3.12
conda activate yolo11env
conda install pip -y
conda install ipykernel -y
python -m ipykernel install --user --name yolo11env --display-name "yolo11env"

pip install ultralytics
pip install azureml-inference-server-http
pip install azure-monitor-opentelemetry-exporter
pip install pip install azure-ai-ml
pip install pip install azure-identity
```

- First is to accept the terms of service for Anaconda packages.
- Next create a new conda environment named yolo11env with python 3.12
- Activate the environment and install pip and ipykernel.
- Install ultralytics package to use yolo 11n model.
- Install azureml-inference-server-http package to create scoring script for Azure ML managed online endpoint.
- Install azure-monitor-opentelemetry-exporter package for monitoring.
- Then install azure-ai-ml and azure-identity packages to register and deploy model.
- Next create new folders one called models and another called src.
- Upload the yolo11n.pt model file from Hugging Face Model Hub to models folder.
- Create the score.py in src folder.
- Next create a new python script named score.py with the following code to load the model and perform inference.

```python
import os
import json
import requests
from ultralytics import YOLO
from PIL import Image
import io

def init():
    """
    Initialize the YOLO model once when the endpoint starts.
    """
    global model
    try:
        model_path = os.path.join(os.getenv("AZUREML_MODEL_DIR"), "yolo11n.pt")
        if not os.path.exists(model_path):
            raise FileNotFoundError(f"Model file not found at {model_path}")
        model = YOLO(model_path)
        print(f"Model loaded from {model_path}")
    except Exception as e:
        print(f"Error loading model: {str(e)}")
        raise

def run(raw_data):
    """
    Run inference on an image URL.
    Expects raw_data as JSON string: {"image_url": "<URL>"}
    Returns JSON dict with YOLO results or error message.
    """
    try:
        # Parse input
        data = json.loads(raw_data)
        if "image_url" not in data:
            return {"error": "Missing 'image_url' in request"}
        image_url = data["image_url"]

        # Download image
        response = requests.get(image_url, timeout=10)
        response.raise_for_status()
        img = Image.open(io.BytesIO(response.content))

        # Run YOLO inference
        results = model(img)
        result = results[0]
        serialized_result = result.to_json()

        return serialized_result

    except requests.exceptions.RequestException as re:
        return {"error": f"Failed to download image: {str(re)}"}
    except Exception as e:
        return {"error": str(e)}
```

- The init() function loads the yolo11n model from the model directory when the endpoint starts.
- The run() function takes input as a JSON string containing an image URL, downloads the image
- performs inference using the yolo11n model, and returns the results as a JSON dict.

## Register and Deploy Model

- We are using AML SDK v2 to register and deploy the model.
- Import necessary packages like os, json, requests, ultralytics YOLO, PIL Image, and io.

```
from azure.ai.ml import MLClient, Input
from azure.ai.ml.entities import (
ManagedOnlineEndpoint,
ManagedOnlineDeployment,
Model,
Environment,
CodeConfiguration,
)
from azure.identity import DefaultAzureCredential
from azure.ai.ml.constants import AssetTypes
```

- Authenticate MLClient using DefaultAzureCredential.

```python
subscription_id = "xxxxxxxxxxxxxxxxxxxxxxxx"
resource_group = "rgname"
workspace = "workspace_name"

ml_client = MLClient(DefaultAzureCredential(), subscription_id, resource_group, workspace)
```

- Now register the model

```
model_name = 'yolo11n-object'
model_local_path = "models"
model = ml_client.models.create_or_update(
        Model(name=model_name, path=model_local_path, type=AssetTypes.CUSTOM_MODEL, description="Yolov11n Model")
)
```

### Managed Online Endpoint and Deployment

- Create Managed Online Endpoint
- Create Endpoint with unique name

```
endpoint_name = "yolo11n-object-20251209-1"
```

- Create the endpoint

```python
import datetime

endpoint = ManagedOnlineEndpoint(
    name=endpoint_name,
    description="An online endpoint to do object detection using yolov11n",
    auth_mode="key",
    tags={"env": "dev"},
)
```

- Create or update the endpoint in AML workspace

```python
poller = ml_client.begin_create_or_update(endpoint)
result = poller.result()   # waits until completion

print("Status:", poller.status())
print("Done:", result)
```

- Now create the environment with required dependencies
- Create Environment from environment.yml

```python
%%writefile environment.yml
name: yolo11env
channels:
  - conda-forge
dependencies:
  - python=3.12
  - pip
  - opencv
  - pip:
    - azureml-inference-server-http
    - inference-schema[numpy-support]
    - joblib
    - Pillow
    - ultralytics
    - azure-monitor-opentelemetry-exporter
```

- Save the above file the root folder.
- Now create the environment in AML workspace
- 
```python
yolo_env = Environment(
    name="yolo11-env",
    image="mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu22.04:latest",
    conda_file="environment.yml"  # <-- now pointing to a real file
)

ml_client.environments.create_or_update(yolo_env)
```

- Now create the blue deployment for the endpoint

```python
from azure.ai.ml.entities import ManagedOnlineDeployment

blue_deployment = ManagedOnlineDeployment(
    name="blue",
    endpoint_name=endpoint_name,
    model=model,
    environment=yolo_env,
    code_path="./src",
    scoring_script="score.py",
    instance_type="Standard_D16a_v4",
    instance_count=1
)
```

- Create or update the deployment

```python
poller = ml_client.online_deployments.begin_create_or_update(blue_deployment)

# This will block until deployment is finished (Succeeded or Failed)
deployment_result = poller.result()

print("Deployment finished!")
print("Deployment name:", deployment_result.name)
print("Provisioning state:", deployment_result.provisioning_state)
```

- Set the traffic to blue deployment

```python
endpoint.traffic = {"blue": 100}
```

- Update the endpoint with traffic settings

```python
# assume ml_client and endpoint object are defined
poller = ml_client.begin_create_or_update(endpoint)

# Wait until the endpoint is fully created
endpoint_result = poller.result()  # blocks here until completion

print("Endpoint creation finished!")
print("Provisioning state:", endpoint_result.provisioning_state)
print("Endpoint name:", endpoint_result.name)
```

- Get logs

```
ml_client.online_deployments.get_logs(
    name="blue",
    endpoint_name=endpoint_name,
    lines=200
)
```

- Test the endpoint with sample input data

```python
payload = {"image_url": "https://ultralytics.com/images/bus.jpg"}

response = ml_client.online_endpoints.invoke(
    endpoint_name=endpoint_name,
    request_body=payload,  # dict instead of JSON string
    content_type="application/json"
)
```

- Done! You have successfully deployed Yolo 11n model in Azure Machine Learning Managed Online Endpoint.