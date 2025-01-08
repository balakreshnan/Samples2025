# Azure Machine learning inferecing with Microsoft Phi 4

## Introduction

- This notebook demonstrates how to deploy a phi 4 model on Azure Machine Learning (AML) using jupyter notebook.
- Huggingface microsoft phi 4 model
- Testing on smaller footprint.

## Prerequisites

- Azure subscription
- Azure Machine Learning workspace
- GPU compute cluster
- Standard_NC96ads_A100_v4 (96 cores, 880 GB RAM, 256 GB disk)
- python 3.12
- Huggingface microsoft phi 4 model

## Steps

### Code

- install the required packages

```
%pip install --upgrade transformers
%pip install "transformers[accelerate]>=4.43.0" --upgrade
%pip install torch
%pip install --upgrade accelerate
```

- log into hugging face

```
from huggingface_hub import notebook_login
notebook_login()
```

- import the required packages

```
import torch
import transformers
```

- if there is no space please delete older hugging face hub

```
/home/azureuser/.cache/huggingface/hub
```

- Load the model

```
import transformers

pipeline = transformers.pipeline(
    "text-generation",
    model="microsoft/phi-4",
    model_kwargs={"torch_dtype": "auto"},
    device_map="auto",
)
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/phi4-3.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/phi4-4.jpg 'RagChat')

- now set the messages

```
messages = [
    {"role": "system", "content": "You are a the greatest chef and must provide explanations to people in simple language."},
    {"role": "user", "content": "Can you create a new veggie burger receipe?"},
]
```

- now generate the response

```
outputs = pipeline(messages, max_new_tokens=500)
print(outputs[0]["generated_text"][-1])
```

```
outputs[0]["generated_text"][-1]["content"]
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/phi4-5.jpg 'RagChat')

- done