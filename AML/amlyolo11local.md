# Deploy Yolo 11n in Azure Machine Learning Local to debug and test

## Introduction

- Ability to deploy Yolo 11n model in Azure Machine Learning Managed Online Endpoint.
- But before that test local deployment in Azure Machine Learning Compute Instance.
- There are 2 main steps to achieve this.
- First is to test with Azure inferencing server. We need the model and scoring file for that.
- Second is to create docker container with environment.yml file and test local deployment. This basically simulates the managed online endpoint deployment.
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

## Docker setup

- install docker desktop in your compute instance.
- Go to compute instance terminal and run the following command to install docker.
- Change the environment to yolo11env if not already activated.

```
conda activate yolo11env
```

```
pip install docker
```

- Now we have to change the configuration of docker to use more resources.
- Go to terminal in compute instance.
- Create a new folder called /mnt/dockerdata
- We are going to use this folder to store docker data.
- We are also going to change the default location of docker data to this folder.
- We are also changing the build to use above folder.

- create directory for docker data

```
sudo mkdir /mnt/dockerdata
```

- Edit the docker daemon.json file

```
sudo vi /etc/docker/daemon.json
``` 

- scroll to last but one line above and click a for append mode.
- Add the following lines

```
{
  "data-root": "/mnt/dockerdata",
}
```

- save the file and exit.
- Restart docker service

```
sudo systemctl restart docker
```

- to change the build location create or edit the .docker/config.json file in home directory

```
echo 'export DOCKER_TMPDIR=/mnt/dockerdata' | sudo tee -a /etc/environment
```

- restart the docker service again

```
sudo systemctl restart docker
```

- Now to clean up the docker system and free up space run the following command

```
docker system prune -a
```

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

## Code to Local deployment to test in terminal using Azure ML Inference Server

- Go to computere instance terminal.
- Activate the yolo11env environment.

```
conda activate yolo11env
```

- We need the model in models folder and scoring script in src folder.
- Now run the following command to start the Azure ML Inference server with the model and scoring script.

```
azmlinfsrv --entry_script ./src/score.py --model_dir ./models
```

- Wait for the server to start.
- Once the server is started you will see a message like below.

```
Azure ML Inferencing HTTP server v1.5.0


Server Settings
---------------
Entry Script Name: /mnt/batch/tasks/shared/LS_root/mounts/clusters/devbox/code/Users/babal/src/score.py
Model Directory: ./models
Config File: None
Worker Count: 1
Worker Timeout (seconds): None
Server Port: 5001
Health Port: 5001
Application Insights Enabled: false
Application Insights Key: None
Inferencing HTTP server version: azmlinfsrv/1.5.0
CORS for the specified origins: None
Create dedicated endpoint for health: None


Server Routes
---------------
Liveness Probe: GET   127.0.0.1:5001/
Score:          POST  127.0.0.1:5001/score

2025-12-10 13:23:08,362 I [13458] gunicorn.error - Starting gunicorn 23.0.0
2025-12-10 13:23:08,362 I [13458] gunicorn.error - Listening at: http://0.0.0.0:5001 (13458)
2025-12-10 13:23:08,362 I [13458] gunicorn.error - Using worker: sync
2025-12-10 13:23:08,365 I [13618] gunicorn.error - Booting worker with pid: 13618
2025-12-10 13:23:47,665 W [13618] azmlinfsrv - Found extra keys in the config file that are not supported by the server.
Extra keys = ['AZUREML_ENTRY_SCRIPT', 'AZUREML_MODEL_DIR']
Initializing logger
2025-12-10 13:23:54,340 I [13618] azmlinfsrv - Starting up app insights client
2025-12-10 13:23:54,470 E [13618] azmlinfsrv.trace - Error while fetching model IDs
Traceback (most recent call last):
  File "/anaconda/envs/yolo11env/lib/python3.12/site-packages/azureml_inference_server_http/server/appinsights_client.py", line 230, in _get_model_ids
    versions = [int(version) for version in os.listdir(os.path.join(config.azureml_model_dir, model))]
                                            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
NotADirectoryError: [Errno 20] Not a directory: './models/.amlignore'
WARNING ⚠️ user config directory '/home/azureuser/.config/Ultralytics' is not writable, using '/tmp/Ultralytics'. Set YOLO_CONFIG_DIR to override.
Creating new Ultralytics Settings v0.0.6 file ✅ 
View Ultralytics Settings with 'yolo settings' or at '/tmp/Ultralytics/settings.json'
Update Settings with 'yolo settings key=value', i.e. 'yolo settings runs_dir=path/to/dir'. For help see https://docs.ultralytics.com/quickstart/#ultralytics-settings.
2025-12-10 13:24:03,792 I [13618] azmlinfsrv.user_script - Found user script at /mnt/batch/tasks/shared/LS_root/mounts/clusters/devbox/code/Users/babal/src/score.py
2025-12-10 13:24:03,792 I [13618] azmlinfsrv.user_script - run() is not decorated. Server will invoke it with the input in JSON string.
2025-12-10 13:24:03,792 I [13618] azmlinfsrv.user_script - Invoking user's init function
2025-12-10 13:24:04,706 I [13618] azmlinfsrv.print - Model loaded from ./models/yolo11n.pt
2025-12-10 13:24:04,706 I [13618] azmlinfsrv.user_script - Users's init has completed successfully
2025-12-10 13:24:04,988 I [13618] azmlinfsrv.swagger - Swaggers are prepared for the following versions: [2, 3, 3.1].
2025-12-10 13:24:04,988 I [13618] azmlinfsrv - Scoring timeout is set to 3600000
2025-12-10 13:24:04,988 I [13618] azmlinfsrv - Worker with pid 13618 ready for serving traffic
```

- Now open another terminal window in the compute instance.
- Activate the yolo11env environment.

```
conda activate yolo11env
```

- now run the following curl command to test the local deployment with a sample image url.

```
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"image_url": "https://ultralytics.com/images/bus.jpg"}' \
     http://127.0.0.1:5001/score
```

- You should see the inference results in JSON format in the terminal.
- This confirms that the local deployment of yolo11n model using Azure ML Inference server
- is successful and working as expected.
- you should see the bounding bozes with labels and confidence scores in the output JSON.

```
"[{\"name\":\"bus\",\"class\":5,\"confidence\":0.94015,\"box\":{\"x1\":3.83272,\"y1\":229.36424,\"x2\":796.19458,\"y2\":728.41229}},{\"name\":\"person\",\"class\":0,\"confidence\":0.88822,\"box\":{\"x1\":671.01721,\"y1\":394.83307,\"x2\":809.80975,\"y2\":878.71246}},{\"name\":\"person\",\"class\":0,\"confidence\":0.87825,\"box\":{\"x1\":47.40473,\"y1\":399.56512,\"x2\":239.30066,\"y2\":904.19501}},{\"name\":\"person\",\"class\":0,\"confidence\":0.85577,\"box\":{\"x1\":223.05899,\"y1\":408.68857,\"x2\":344.46762,\"y2\":860.43573}},{\"name\":\"person\",\"class\":0,\"confidence\":0.62192,\"box\":{\"x1\":0.02171,\"y1\":556.06854,\"x2\":68.88546,\"y2\":872.35919}}]"
```

- Now it shows the inference results with detected objects in the image along with their bounding boxes and confidence scores.
- This confirms that the local deployment of yolo11n model using Azure ML Inference server is successful and working as expected.

## Local deployment using Docker

- Now import the libraries.

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

- Setup the workspace details.

```
subscription_id = "xxxxxxxxxxxxxxxxxxxxxxx"
resource_group = "rgname"
workspace = "workspace_name"
```

- Now authenticate MLClient using DefaultAzureCredential.

```python
credential = DefaultAzureCredential()
ml_client = MLClient(
    credential,
    subscription_id=subscription_id,
    resource_group_name=resource_group,
    workspace_name=workspace,
)
```

- setup endpoint name

```
endpoint_name = "yolo11n-object-local"
```

- Deploy the model using docker container in local machine.

```python
deployment = ManagedOnlineDeployment(
    name="blue",
    endpoint_name=endpoint_name,
    model=Model(path="./models/yolo11n.pt"),
    code_configuration=CodeConfiguration(
        code="./src", scoring_script="score.py"
    ),
    environment=Environment(
        conda_file="./environment.yml",
        image="mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu22.04:latest",
    ),
    instance_type="STANDARD_DS5_V2",
    instance_count=1,
)

deployment = ml_client.online_deployments.begin_create_or_update(
    deployment, local=True, vscode_debug=False
)
```

- getting logs

```
# Assuming you already have ml_client
# endpoint_name = "yolo11n-object-local"
deployment_name = "blue"

# Fetch logs for the failed deployment
logs = ml_client.online_deployments.get_logs(
    endpoint_name=endpoint_name,
    deployment_name=deployment_name,
    local=True  # important for local endpoints
)

print(logs)
```

- Now test the local endpoint using curl command

```
# Correct: use request_body for dicts, do NOT use request_file
response = ml_client.online_endpoints.invoke(
    endpoint_name=endpoint_name,
    request_body=payload,  # pass the dict here
    local=True             # run against local deployment
)

# Print nicely
print(json.dumps(response, indent=2))
```

- You should see the inference results in JSON format in the terminal.
- This confirms that the local deployment of yolo11n model using Azure ML Managed Online Endpoint is successful and working as expected.