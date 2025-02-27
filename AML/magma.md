# Microsoft Magma model in Industrial Use Cases

## Introduction

- Microsoft Magma is a new model that has been developed by Microsoft Research.
- It is a foundation model for multimodal AI agents.
- Idea here is to use in Industrial scenarios to detect safety violations in images.
- Also can be used in Healthcare, Retail, Manufacturing, Construction, etc.

## Pre-requisites

- Azure Subscription
- Azure machine learning workspace
- GPU compute - Standard_NC96ads_A100_v4 (96 cores, 880 GB RAM, 256 GB disk)
- Python 3.12 and above

## Steps

- First updatede Transformers

```
pip install -U transformers
```

- Clear cache if hard disk is full

```
!rm -rf ~/.cache/huggingface/hub
```

- Now import all the libraries

```
from PIL import Image
import torch
from transformers import AutoModelForCausalLM
from transformers import AutoProcessor
import urllib.request
from io import BytesIO
```

- Download the model

```
dtype = torch.bfloat16
model = AutoModelForCausalLM.from_pretrained("microsoft/Magma-8B", trust_remote_code=True, torch_dtype=dtype)
processor = AutoProcessor.from_pretrained("microsoft/Magma-8B", trust_remote_code=True)
model.to("cuda")
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/magma-1.jpg 'RagChat')

- import for display image

```
import matplotlib.pyplot as plt
import numpy as np
from PIL import Image
from IPython.display import display
```

- now load the image to process

```
image = Image.open("hard-hat-heroes.jpg").convert("RGB")
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/magma-3.jpg 'RagChat')

- now set the question and prompt

```
convs = [
    {"role": "system", "content": "You are agent that can see, talk and act."},            
    {"role": "user", "content": "<image_start><image><image_end>\find if workers are wearing hard hats, vest, safety googles, gloves?"},
]
prompt = processor.tokenizer.apply_chat_template(convs, tokenize=False, add_generation_prompt=True)
inputs = processor(images=[image], texts=prompt, return_tensors="pt")
inputs['pixel_values'] = inputs['pixel_values'].unsqueeze(0)
inputs['image_sizes'] = inputs['image_sizes'].unsqueeze(0)
inputs = inputs.to("cuda").to(dtype)
```

- now inference the model

```
with torch.inference_mode():
    generate_ids = model.generate(**inputs, **generation_args)

generate_ids = generate_ids[:, inputs["input_ids"].shape[-1] :]
response = processor.decode(generate_ids[0], skip_special_tokens=True).strip()
```

- now display the output

```
print(response)
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/magma-2.jpg 'RagChat')