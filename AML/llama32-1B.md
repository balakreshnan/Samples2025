# Azure Machine learning inferecing with llama3.2 1B

## Introduction

- This notebook demonstrates how to deploy a llama3.2 1B model on Azure Machine Learning (AML) using jupyter notebook.
- Huggingface llama3.2 1B model
- Testing on smaller footprint.

## Prerequisites

- Azure subscription
- Azure Machine Learning workspace
- GPU compute cluster
- Standard_NC64as_T4_v3 (64 cores, 440 GB RAM, 2816 GB disk)
- python 3.12
- Huggingface llama3.2 1B model

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
from transformers import pipeline
```

- configure the Model

```
model_id = "meta-llama/Llama-3.2-1B"
```

- Load the model

```
pipe = pipeline(
    "text-generation", 
    model=model_id, 
    torch_dtype=torch.bfloat16, 
    device_map="auto"
)
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama32-1b-1.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama32-1b-2.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama32-1b-3.jpg 'RagChat')

- Test the model

```
response = pipe("Explain Quantum computing with details")
```

- Print the response

```
print(response[0]['generated_text'])
```

- output

```
Explain Quantum computing with details
The quantum computer can perform the calculation with the help of quantum bits (qubits) that can
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama32-1b-4.jpg 'RagChat')