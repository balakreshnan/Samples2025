# MobileLLM-125M - Running in Azure Machine Learning compute

## Introduction

- MobileLLM-125M running in Azure Machine Learning compute.
- using nc48s_v3 (48 vCPUs, 192 GiB memory) instance.
- Small language model for mobile devices.
- Only for educational purposes.

## Pre-requisites

- Azure subscription.
- Azure Machine Learning workspace.
- Hugging Face account.
- access to facebook/MobileLLM-125M

## Steps

- install libraries.

```
%pip install transformers SentencePiece flash_attn
```

- download the model.

```
from transformers import AutoModelForCausalLM, AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained("facebook/MobileLLM-125M", use_fast=False)
model = AutoModelForCausalLM.from_pretrained("facebook/MobileLLM-125M", trust_remote_code=True)
```

- set the message

```
messages = [{"role": "user", "content": "Write a story about agentic AI economy and what impact will it have with humans."}]
```

- generate the response

```
input_text=tokenizer.apply_chat_template(messages, tokenize=False)
inputs = tokenizer.encode(input_text, return_tensors="pt").to(device)
outputs = model.generate(inputs, max_new_tokens=2000, temperature=0.2, top_p=0.9, do_sample=True)
print(tokenizer.decode(outputs[0]))
```

- output

```
```

- more to come