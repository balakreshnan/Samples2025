# Fine tuning AOAIFineTuning gpt4o-mini

## Introduction

- Show the process of fine-tuning AOAIFineTuning gpt4o-mini
- AOAI fine tuning is a process of training a pre-trained model on a specific dataset to improve its performance on a specific task.
- using the sample data provided by documentation - https://learn.microsoft.com/en-us/azure/ai-services/openai/tutorials/fine-tune?tabs=command-line

## Prerequisites

- Azure subscription
- Azure AI foundry
- Deploy gpt 4o-mini model
- Pick the region EastUS2
- Visual Studio Code
- python 3.13 or later

## Code

- import libraries

```
import json
import os
import requests
import tiktoken
import numpy as np
from collections import defaultdict
from openai import AzureOpenAI
from IPython.display import clear_output
import time

encoding = tiktoken.get_encoding("o200k_base") # default encoding for gpt-4o models. This requires the latest version of tiktoken to be installed.
```

- invoke the Azure openai model

```
client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"),
  api_key = os.getenv("AZURE_OPENAI_API_KEY"),
  api_version = "2025-02-01-preview"  
)
```

- now create a function to get the number of tokens in a string

```
def Checkdata():
    # Check if the required environment variables are set
    try:
        # Load the training set
        with open('training_set.jsonl', 'r', encoding='utf-8') as f:
            training_dataset = [json.loads(line) for line in f]

        # Training dataset stats
        print("Number of examples in training set:", len(training_dataset))
        print("First example in training set:")
        for message in training_dataset[0]["messages"]:
            print(message)

        # Load the validation set
        with open('validation_set.jsonl', 'r', encoding='utf-8') as f:
            validation_dataset = [json.loads(line) for line in f]

        # Validation dataset stats
        print("\nNumber of examples in validation set:", len(validation_dataset))
        print("First example in validation set:")
        for message in validation_dataset[0]["messages"]:
            print(message)
        

    except AssertionError as e:
        print(f"Environment variable check failed: {e}")
        return


def num_tokens_from_messages(messages, tokens_per_message=3, tokens_per_name=1):
    num_tokens = 0
    for message in messages:
        num_tokens += tokens_per_message
        for key, value in message.items():
            num_tokens += len(encoding.encode(value))
            if key == "name":
                num_tokens += tokens_per_name
    num_tokens += 3
    return num_tokens

def num_assistant_tokens_from_messages(messages):
    num_tokens = 0
    for message in messages:
        if message["role"] == "assistant":
            num_tokens += len(encoding.encode(message["content"]))
    return num_tokens

def print_distribution(values, name):
    print(f"\n#### Distribution of {name}:")
    print(f"min / max: {min(values)}, {max(values)}")
    print(f"mean / median: {np.mean(values)}, {np.median(values)}")
    print(f"p5 / p95: {np.quantile(values, 0.1)}, {np.quantile(values, 0.9)}")

def processtoken():
    files = ['training_set.jsonl', 'validation_set.jsonl']

    for file in files:
        print(f"Processing file: {file}")
        with open(file, 'r', encoding='utf-8') as f:
            dataset = [json.loads(line) for line in f]

        total_tokens = []
        assistant_tokens = []

        for ex in dataset:
            messages = ex.get("messages", {})
            total_tokens.append(num_tokens_from_messages(messages))
            assistant_tokens.append(num_assistant_tokens_from_messages(messages))

        print_distribution(total_tokens, "total tokens")
        print_distribution(assistant_tokens, "assistant tokens")
        print('*' * 50)

def uploadfinetunefiles():
    training_file_name = 'training_set.jsonl'
    validation_file_name = 'validation_set.jsonl'

    # Upload the training and validation dataset files to Azure OpenAI with the SDK.

    training_response = client.files.create(
        file = open(training_file_name, "rb"), purpose="fine-tune"
    )
    training_file_id = training_response.id

    validation_response = client.files.create(
        file = open(validation_file_name, "rb"), purpose="fine-tune"
    )
    validation_file_id = validation_response.id

    print("Training file ID:", training_file_id)
    print("Validation file ID:", validation_file_id)

def deploymodel():
    # Deploy fine-tuned model


    token = os.getenv("TEMP_AUTH_TOKEN")
    subscription = os.getenv("AZURE_SUBSCRIPTION_ID")
    resource_group = os.getenv("AZURE_RESOURCE_GROUP")
    resource_name = os.getenv("AZURE_OPENAI_RESOURCE_NAME")
    model_deployment_name = "gpt-4o-mini-2024-07-18-ft" # Custom deployment name you chose for your fine-tuning model

    deploy_params = {'api-version': "2024-10-01"} # Control plane API version
    deploy_headers = {'Authorization': 'Bearer {}'.format(token), 'Content-Type': 'application/json'}

    deploy_data = {
        "sku": {"name": "standard", "capacity": 1},
        "properties": {
            "model": {
                "format": "OpenAI",
                "name": "<YOUR_FINE_TUNED_MODEL>", #retrieve this value from the previous call, it will look like gpt-4o-mini-2024-07-18.ft-0e208cf33a6a466994aff31a08aba678
                "version": "1"
            }
        }
    }
    deploy_data = json.dumps(deploy_data)

    request_url = f'https://management.azure.com/subscriptions/{subscription}/resourceGroups/{resource_group}/providers/Microsoft.CognitiveServices/accounts/{resource_name}/deployments/{model_deployment_name}'

    print('Creating a new deployment...')

    r = requests.put(request_url, params=deploy_params, headers=deploy_headers, data=deploy_data)

    print(r)
    print(r.reason)
    print(r.json())

def deploytest():
    returntxt  = ""
    response = client.chat.completions.create(
        model = "gpt-4o-mini-2024-07-18-ft", # model = "Custom deployment name you chose for your fine-tuning model"
        messages = [
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Does Azure OpenAI support customer managed keys?"},
            {"role": "assistant", "content": "Yes, customer managed keys are supported by Azure OpenAI."},
            {"role": "user", "content": "Do other Azure services support this too?"}
        ]
    )

    print(response.choices[0].message.content)

    returntxt = response.choices[0].message.content
    return returntxt
```

- now create the main function to load training and validation data, upload the files to Azure OpenAI, deploy the model, and test the deployment

```
def main():
    # Check if the required environment variables are set
    Checkdata()

    # Process the training and validation datasets
    processtoken()

    # Upload the training and validation datasets to Azure OpenAI
    training_file_name = 'training_set.jsonl'
    validation_file_name = 'validation_set.jsonl'

    # Upload the training and validation dataset files to Azure OpenAI with the SDK.

    training_response = client.files.create(
        file = open(training_file_name, "rb"), purpose="fine-tune"
    )
    training_file_id = training_response.id

    validation_response = client.files.create(
        file = open(validation_file_name, "rb"), purpose="fine-tune"
    )
    validation_file_id = validation_response.id

    while True:
        file_status = client.files.retrieve(training_file_id).status
        if file_status == "processed":
            break
        time.sleep(10)  # Check every 10 seconds

    while True:
        file_status = client.files.retrieve(validation_file_id).status
        if file_status == "processed":
            break
        time.sleep(10)  # Check every 10 seconds

    file_status = client.files.retrieve(training_file_id).status
    print(f"Training File status: {file_status}")  # Should return "processed"

    file_status = client.files.retrieve(validation_file_id).status
    print(f"Validation File status: {file_status}")  # Should return "processed"


    print("Training file ID:", training_file_id)
    print("Validation file ID:", validation_file_id)
    # Submit fine-tuning training job

    response = client.fine_tuning.jobs.create(
        training_file = training_file_id,
        validation_file = validation_file_id,
        model = "gpt-4o-mini-2024-07-18", # Enter base model name. Note that in Azure OpenAI the model name contains dashes and cannot contain dot/period characters.
        seed = 105 # seed parameter controls reproducibility of the fine-tuning job. If no seed is specified one will be generated automatically.
    )

    job_id = response.id

    # You can use the job ID to monitor the status of the fine-tuning job.
    # The fine-tuning job will take some time to start and complete.

    print("Job ID:", response.id)
    print("Status:", response.status)
    print(response.model_dump_json(indent=2))

    # Monitor the fine-tuning job status
    start_time = time.time()

    # Get the status of our fine-tuning job.
    response = client.fine_tuning.jobs.retrieve(job_id)

    status = response.status

    # If the job isn't done yet, poll it every 10 seconds.
    while status not in ["succeeded", "failed"]:
        time.sleep(10)

        response = client.fine_tuning.jobs.retrieve(job_id)
        print(response.model_dump_json(indent=2))
        print("Elapsed time: {} minutes {} seconds".format(int((time.time() - start_time) // 60), int((time.time() - start_time) % 60)))
        status = response.status
        print(f'Status: {status}')
        clear_output(wait=True)

    print(f'Fine-tuning job {job_id} finished with status: {status}')

    # List all fine-tuning jobs for this resource.
    print('Checking other fine-tune jobs for this resource.')
    response = client.fine_tuning.jobs.list()
    print(f'Found {len(response.data)} fine-tune jobs.')

    response = client.fine_tuning.jobs.list_events(fine_tuning_job_id=job_id, limit=10)
    print(response.model_dump_json(indent=2))

    response = client.fine_tuning.jobs.checkpoints.list(job_id)
    print(response.model_dump_json(indent=2))

    # Retrieve fine_tuned_model name

    response = client.fine_tuning.jobs.retrieve(job_id)

    print(response.model_dump_json(indent=2))
    fine_tuned_model = response.fine_tuned_model

    # for deployment
    # https://learn.microsoft.com/en-us/azure/ai-services/openai/tutorials/fine-tune?tabs=command-line
```

- now invoke the main function

```
if __name__ == "__main__":
    main()
```

- now run the code in the terminal

```
python fine_tune.py
```

- wait for the fine-tuning job to complete. This may take some time depending on the size of your dataset and the model you are using.
- it took about 1 hour to complete the fine-tuning job.
- Here is the output in the terminal

```
{
  "id": "ftjob-41e7d3a9b59541148bc5514c3b74e616",
  "created_at": 1748019281,
  "error": null,
  "fine_tuned_model": "gpt-4o-mini-2024-07-18.ft-41e7d3a9b59541148bc5514c3b74e616",
  "finished_at": 1748023247,
  "hyperparameters": {
    "batch_size": 1,
    "learning_rate_multiplier": 1.0,
    "n_epochs": 10
  },
  "model": "gpt-4o-mini-2024-07-18",
  "object": "fine_tuning.job",
  "organization_id": null,
  "result_files": [
    "file-4b26d4b72321456b9589282ecc6df9bd"
  ],
  "seed": 105,
  "status": "succeeded",
  "trained_tokens": 6720,
  "training_file": "file-5ee7a60743fb4f6886a005333bdfed4b",
  "validation_file": "file-304c29d3c9264f96abd47b9aaf4aa8bc",
  "estimated_finish": 1748022078,
  "integrations": null,
  "metadata": null,
  "method": null
}
Elapsed time: 66 minutes 12 seconds
Status: succeeded
Fine-tuning job ftjob-41e7d3a9b59541148bc5514c3b74e616 finished with status: succeeded
Checking other fine-tune jobs for this resource.
Found 1 fine-tune jobs.
{
  "data": [
    {
      "id": "ftevent-49e713eebeda4aeababd7e3b8fbcaf95",
      "created_at": 1748023247,
      "level": "info",
      "message": "Training tokens billed: 5000",
      "object": "fine_tuning.job.event",
      "data": null,
      "type": "message"
    },
    {
      "id": "ftevent-8d595a34ae2643c296334cbe4b8b85ee",
      "created_at": 1748023247,
      "level": "info",
      "message": "Completed results file: file-4b26d4b72321456b9589282ecc6df9bd",
      "object": "fine_tuning.job.event",
      "data": null,
      "type": "message"
    },
    {
      "id": "ftevent-d746504945ad49f29942416b8f56036a",
      "created_at": 1748023246,
      "level": "info",
      "message": "Model Evaluation Passed.",
      "object": "fine_tuning.job.event",
      "data": null,
      "type": "message"
    },
    {
      "id": "ftevent-526635aaf862416da867129cc67ddd23",
      "created_at": 1748023018,
      "level": "info",
      "message": "Job succeeded.",
      "object": "fine_tuning.job.event",
      "data": null,
      "type": "message"
    },
    {
      "id": "ftevent-008dd9a2143e297008dd9a2143e29700",
      "created_at": 1748022182,
      "level": "info",
      "message": "Step 100: training loss=0.0007721185684204102",
      "object": "fine_tuning.job.event",
      "data": {
        "step": 100,
        "train_loss": 0.0007721185684204102,
        "train_mean_token_accuracy": 1,
        "valid_loss": 1.5302696228027344,
        "valid_mean_token_accuracy": 0.7857142857142857,
        "full_valid_loss": 2.278649262820973,
        "full_valid_mean_token_accuracy": 0.7176470588235294
      },
      "type": "metrics"
    },
    {
      "id": "ftevent-008dd9a213decb6008dd9a213decb600",
      "created_at": 1748022172,
      "level": "info",
      "message": "Step 90: training loss=0.00025653839111328125",
      "object": "fine_tuning.job.event",
      "data": {
        "step": 90,
        "train_loss": 0.00025653839111328125,
        "train_mean_token_accuracy": 1,
        "valid_loss": 1.1953933715820313,
        "valid_mean_token_accuracy": 0.8666666666666667,
        "full_valid_loss": 2.251041165520163,
        "full_valid_mean_token_accuracy": 0.7058823529411765
      },
      "type": "metrics"
    },
    {
      "id": "ftevent-008dd9a2137f6d5008dd9a2137f6d500",
      "created_at": 1748022162,
      "level": "info",
      "message": "Step 80: training loss=0.0020896196365356445",
      "object": "fine_tuning.job.event",
      "data": {
        "step": 80,
        "train_loss": 0.0020896196365356445,
        "train_mean_token_accuracy": 1,
        "valid_loss": 2.0088092402407995,
        "valid_mean_token_accuracy": 0.7368421052631579,
        "full_valid_loss": 2.1294603796566234,
        "full_valid_mean_token_accuracy": 0.7235294117647059
      },
      "type": "metrics"
    },
    {
      "id": "ftevent-008dd9a213200f4008dd9a213200f400",
      "created_at": 1748022152,
      "level": "info",
      "message": "Step 70: training loss=0.007328391075134277",
      "object": "fine_tuning.job.event",
      "data": {
        "step": 70,
        "train_loss": 0.007328391075134277,
        "train_mean_token_accuracy": 1,
        "valid_loss": 1.9906560807001024,
        "valid_mean_token_accuracy": 0.7142857142857143,
        "full_valid_loss": 1.8927684222950656,
        "full_valid_mean_token_accuracy": 0.7294117647058823
      },
      "type": "metrics"
    },
    {
      "id": "ftevent-008dd9a212c0b13008dd9a212c0b1300",
      "created_at": 1748022142,
      "level": "info",
      "message": "Step 60: training loss=0.0028086344245821238",
      "object": "fine_tuning.job.event",
      "data": {
        "step": 60,
        "train_loss": 0.0028086344245821238,
        "train_mean_token_accuracy": 1,
        "valid_loss": 2.196942138671875,
        "valid_mean_token_accuracy": 0.7,
        "full_valid_loss": 1.7032222635605756,
        "full_valid_mean_token_accuracy": 0.7235294117647059
      },
      "type": "metrics"
    },
    {
      "id": "ftevent-008dd9a21261532008dd9a2126153200",
      "created_at": 1748022132,
      "level": "info",
      "message": "Step 50: training loss=0.18988676369190216",
      "object": "fine_tuning.job.event",
      "data": {
        "step": 50,
        "train_loss": 0.18988676369190216,
        "train_mean_token_accuracy": 0.949999988079071,
        "valid_loss": 2.155858591983193,
        "valid_mean_token_accuracy": 0.6842105263157895,
        "full_valid_loss": 1.4184704051298254,
        "full_valid_mean_token_accuracy": 0.7470588235294118
      },
      "type": "metrics"
    }
  ],
  "has_more": true,
  "object": "list"
}
{
  "data": [
    {
      "id": "ftchkpt-11e6a8bda669467f9c725a13135e7257",
      "created_at": 1748022569,
      "fine_tuned_model_checkpoint": "gpt-4o-mini-2024-07-18.ft-41e7d3a9b59541148bc5514c3b74e616",
      "fine_tuning_job_id": "ftjob-41e7d3a9b59541148bc5514c3b74e616",
      "metrics": {
        "full_valid_loss": 2.278649262820973,
        "full_valid_mean_token_accuracy": 0.7176470588235294,
        "step": 100.0,
        "train_loss": 0.0007721185684204102,
        "train_mean_token_accuracy": 1.0,
        "valid_loss": 1.5302696228027344,
        "valid_mean_token_accuracy": 0.7857142857142857
      },
      "object": "fine_tuning.job.checkpoint",
      "step_number": 100
    },
    {
      "id": "ftchkpt-9d4a378428cf417fb7866bdae14c68d9",
      "created_at": 1748022516,
      "fine_tuned_model_checkpoint": "gpt-4o-mini-2024-07-18.ft-41e7d3a9b59541148bc5514c3b74e616:ckpt-step-90",
      "fine_tuning_job_id": "ftjob-41e7d3a9b59541148bc5514c3b74e616",
      "metrics": {
        "full_valid_loss": 2.251041165520163,
        "full_valid_mean_token_accuracy": 0.7058823529411765,
        "step": 90.0,
        "train_loss": 0.00025653839111328125,
        "train_mean_token_accuracy": 1.0,
        "valid_loss": 1.1953933715820313,
        "valid_mean_token_accuracy": 0.8666666666666667
      },
      "object": "fine_tuning.job.checkpoint",
      "step_number": 90
    },
    {
      "id": "ftchkpt-99488d8ff9a64ce38b6dfb44934fb626",
      "created_at": 1748022462,
      "fine_tuned_model_checkpoint": "gpt-4o-mini-2024-07-18.ft-41e7d3a9b59541148bc5514c3b74e616:ckpt-step-80",
      "fine_tuning_job_id": "ftjob-41e7d3a9b59541148bc5514c3b74e616",
      "metrics": {
        "full_valid_loss": 2.1294603796566234,
        "full_valid_mean_token_accuracy": 0.7235294117647059,
        "step": 80.0,
        "train_loss": 0.0020896196365356445,
        "train_mean_token_accuracy": 1.0,
        "valid_loss": 2.0088092402407995,
        "valid_mean_token_accuracy": 0.7368421052631579
      },
      "object": "fine_tuning.job.checkpoint",
      "step_number": 80
    }
  ],
  "has_more": false,
  "object": "list"
}
{
  "id": "ftjob-41e7d3a9b59541148bc5514c3b74e616",
  "created_at": 1748019281,
  "error": null,
  "fine_tuned_model": "gpt-4o-mini-2024-07-18.ft-41e7d3a9b59541148bc5514c3b74e616",
  "finished_at": 1748023247,
  "hyperparameters": {
    "batch_size": 1,
    "learning_rate_multiplier": 1.0,
    "n_epochs": 10
  },
  "model": "gpt-4o-mini-2024-07-18",
  "object": "fine_tuning.job",
  "organization_id": null,
  "result_files": [
    "file-4b26d4b72321456b9589282ecc6df9bd"
  ],
  "seed": 105,
  "status": "succeeded",
  "trained_tokens": 6720,
  "training_file": "file-5ee7a60743fb4f6886a005333bdfed4b",
  "validation_file": "file-304c29d3c9264f96abd47b9aaf4aa8bc",
  "estimated_finish": 1748022078,
  "integrations": null,
  "metadata": null,
  "method": null
}
```

- now go to Azure AI foundry and check the deployment status

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aoaifinetune-1.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aoaifinetune-2.jpg 'RagChat')

- Metrics

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aoaifinetune-3.jpg 'RagChat')

- Check point

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aoaifinetune-4.jpg 'RagChat')

- Now we can do continual deployment of the model
- Deploy the model for inferencing.
- Download the results file and training and validation file.
- Delete the training and validation files.