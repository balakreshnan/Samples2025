# Azure Machine learning inferecing with llama3.170bint4

## Introduction

- This notebook demonstrates how to deploy a llama3.170bint4 model on Azure Machine Learning (AML) using Azure Kubernetes Service (AKS) as the compute target.
- Quantized model from hugging face.
- Testing on smaller footprint.

## Prerequisites

- Azure subscription
- Azure Machine Learning workspace
- GPU compute cluster
- SKU - Standard_NC96ads_A100_v4 (96 cores, 880 GB RAM, 256 GB disk)
- python 3.12
- Huggingface quantized model

## Steps

### Code

- install the required packages

```
%pip install --upgrade transformers
%pip install "transformers[accelerate]>=4.43.0" --upgrade
%pip install torch
%pip install ipywidgets
%pip install --upgrade autoawq
%pip install autoawq[kernels]
%pip install --upgrade accelerate
```

- Here is the actual code
- now import the required packages

```
import transformers
import torch
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer, AwqConfig
```

- Load the model and tokenizer

```
model_id = "hugging-quants/Meta-Llama-3.1-70B-Instruct-AWQ-INT4"
quantization_config = AwqConfig(
    bits=4,
    fuse_max_seq_len=512, # Note: Update this as per your use-case
    do_fuse=True,
)
```

- now load the tokenizer and model

```
tokenizer = AutoTokenizer.from_pretrained(model_id)
model = AutoModelForCausalLM.from_pretrained(
  model_id,
  torch_dtype=torch.float16,
  low_cpu_mem_usage=True,
  device_map="auto",
  quantization_config=quantization_config
)
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama31-70-int4-4.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama3.1-70Int4-1.jpg 'RagChat')

- create the message

```
prompt = [
  {"role": "system", "content": "You are a helpful assistant, that responds as a pirate."},
  {"role": "user", "content": "What's Deep Learning?"},
]
```

- setup the model for inference

```
inputs = tokenizer.apply_chat_template(
  prompt,
  tokenize=True,
  add_generation_prompt=True,
  return_tensors="pt",
  return_dict=True,
).to("cuda")
```

- now generate the response

```
outputs = model.generate(**inputs, do_sample=True, max_new_tokens=256)
print(tokenizer.batch_decode(outputs[:, inputs['input_ids'].shape[1]:], skip_special_tokens=True)[0])
```

- output

```
Deep Learning!

Deep Learning be a subset o' Machine Learning, which be a part o' Artificial Intelligence (AI). It be a type o' neural network that be inspired by the structure o' the human brain.

Deep Learning be used fer a variety o' tasks, such as:

* Image recognition: Deep Learning be used fer image recognition tasks, such as identifying objects in images.
* Speech recognition: Deep Learning be used fer speech recognition tasks, such as transcribing spoken words into text.
* Natural Language Processing (NLP): Deep Learning be used fer NLP tasks, such as text classification, sentiment analysis, and machine translation.

So hoist the sails and set course fer the world o' Deep Learning, me hearty!
```


![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama3.1-70Int4-2.jpg 'RagChat')