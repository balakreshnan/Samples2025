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
- make sure all libraries are latest.

```
%pip install transformers SentencePiece flash_attn
```

- download the model.

```
from transformers import AutoModelForCausalLM, AutoTokenizer
tokenizer = AutoTokenizer.from_pretrained("facebook/MobileLLM-125M", use_fast=False)
model = AutoModelForCausalLM.from_pretrained("facebook/MobileLLM-125M", trust_remote_code=True)
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/mobilellm125-1.jpg 'RagChat')

- set the message

```
import torch
# Define the input prompt
prompt = "Write a story about agentic AI economy and what impact will it have with humans."

# Tokenize the input
inputs = tokenizer(prompt, return_tensors="pt")

# Generate output using the model
with torch.no_grad():
    outputs = model.generate(
        inputs["input_ids"],
        max_length=2000,  # Adjust max_length as needed
        num_return_sequences=1,
        temperature=0.7  # Adjust temperature for creativity in the output
    )

# Decode and print the generated text
generated_text = tokenizer.decode(outputs[0], skip_special_tokens=True)
print("Generated text:", generated_text)
```

- processing

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/mobilellm125-2.jpg 'RagChat')

- output

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/mobilellm125-3.jpg 'RagChat')

```
Generated text: Write a story about agentic AI economy and what impact will it have with humans.neknekneknek badly divers tags Math tagsnek tagsnekneknekneknek tagsnek tags Como tags tags tags tags tagsний СоUL Palall Pala� server serverнийнийнийzin server server BonTrace Niknek server Gun Math Stadium Stadium Stadiumingo divers tags tags VIIInek Stadium Stadium Akadem affect Stadium highwaylitingo necessity necessity necessityingoingo necessity necessity necessity necessityпример diversottom Bongeometryingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingo tagsingo employees employees ÚingoingoingoMillingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoпримерingoingoingoingoingoingoленняingoingo forceingoingoingoingoingoпримерingoblatt Referencias being Bez tagsingoingoIF Pérez Chiesa Tele finish finishMvcULL tid tagsingoingo necessity necessity tagsingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoIFingoingoingoIFingoingoingoIFIFingoingoingoIFingoingoingoIFIFingoingoingoIFingoingoingoIFIFingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoIFingoingoingoIFIFingoingoingoIFIFingoingo Natalingoingoingoingoingoingo NatalingoingoingoIFIFingoességingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoIF BacingoingoingoingoingoIFingoingoingoingoIFingoingoIFingoingoIFingoingoingoIFingoingoIFingoingo Bacingoјеingoingoingo MediaingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoingoIFingoingoingoPoolingoingoingo moltoingoingounningingoingoingoingoingoingoingoingoingoingoingoingoingo Referenciasingo employees employees employees Referenciasingoingo Referenciasingo employees employees employeesipp Referencias Baby Math employeesdu employees employees employees employees employeesSessioningo СоingoingoingoingoingoingoingoIFingoingoIFingoingoIFingoingo employees employeesinchingoingo employees employees Боingoingoingoingo Math Mathingo Math Akademingoingoingoingoingoingo employeesјеingo employees employees employeesFALSE employees employees employeesippippingoingoingoingo employees finishingo employees employees Mathingoingoingo employeesingo employeesinchcommand Akademingoingoingoligeingoingoingoingoingoligeingoingo finish finishingoingoingoingoingoingoingoingoingoingoIFingoingo employeesingo employeesingo employeesingocommandFALSEingoingo Ker finishFALSE Akadem Akadem Akadem Palaingo Pala Pala Pala Pala finish finish finish finish 
```

- Seems like large text generation seems an issue.
- Tried with max length as 500 and same issue.