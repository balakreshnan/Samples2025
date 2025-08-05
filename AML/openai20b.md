# Inferencing Open AI open source 20B model on Azure ML

## Pre-requisites

- Azure subscription
- Azure ML workspace
- Python 3.12 installed
- GPU SKU: Standard_NC96ads_A100_v4 (96 cores, 880 GB RAM, 256 GB disk)
- 4 GPU A100 in West us3 region
- Using model from Hugging Face: `openai/gpt-oss-20b`
- Hugging Face only has sample code in colab notebook, So we will adapt it to run on AML
- Truly a open source platform

## Steps

- Make sure you have huggingface_hub installed
- Get the API key from Hugging Face
- Now Lets install necessary libraries

```
%pip install -U transformers kernels torch
```

- Now lets import necessary libraries

```
from transformers import pipeline
import torch
```

- Lets login with huggingface

```
from huggingface_hub import notebook_login

notebook_login()
```

- Now lets load the model

```
model_id = "openai/gpt-oss-20b"

pipe = pipeline(
    "text-generation",
    model=model_id,
    torch_dtype="auto",
    device_map="auto",
)
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/openaiosmodel20b-1.jpg 'RagChat')

- Here is the code to generate text

```
messages = [
    {"role": "user", "content": "who won the 1983 cricket world cup? and show me the highlights."},
]

outputs = pipe(
    messages,
    max_new_tokens=256,
)
print(outputs[0]["generated_text"][-1])
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/openaiosmodel20b-2.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/openaiosmodel20b-3.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/openaiosmodel20b-4.jpg 'RagChat')

- As you can see we can run the open source 20B model on Azure ML with GPU SKU.
- Try with more prompts and see the results.
- Will Try fine tuning next time.
- Have fun.