# Fine Tuning qwen 2.5 1.5B Model with Azure Machine Learning - Open Source - with Help Steer Dataset

## Introduction

- Fine tune qwen 2.5 1.5B model with Azure Machine Learning.
- Using HelpSteer dataset as an example.
- To show how to use open source models to fine tune with Azure Machine Learning.
- This guide assumes you have a basic understanding of Python and Azure Machine Learning.
- Only for educational purposes.
- Using Azure Machine Learning to fine tune the model.
- I am using Python 3.12
- Create a separate environment for this project.

```
conda create --name py312
conda activate py312
conda install python==3.12
conda install ipykernel
python -m ipykernel install --user --name py312 --display-name "Python (py312)"
```

## Code to convert HelpSteer dataset to fine-tuning format

- install the required packages

```
pip install torch transformers peft bitsandbytes datasets trl accelerate
pip install huggingface_hub
pip install ipywidgets
```

- List the hugging face cache

```
!ls ~/.cache/huggingface/hub
```

- Clear the existing cache if needed

```
!rm -rf ~/.cache/huggingface/hub
```

- Given these models are huge, AML compute will run out of space.
- Import the required libraries

```
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig, TrainingArguments
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from datasets import Dataset
from trl import SFTTrainer
from huggingface_hub import login
import pandas as pd
import os
```

- Login to Hugging Face
- Need write API key to upload the model after fine-tuning.

```
from huggingface_hub import notebook_login

notebook_login()
```

- Load the dataset

```
from datasets import load_dataset

ds = load_dataset("nvidia/HelpSteer")
```

- print the dataset to see the structure

```
print(ds)
```

- Set the dataset to the training and validation splits

```
train_dataset = ds["train"]
val_dataset = ds["validation"]
```

- Now configure the model to use

```
model_name = "Qwen/Qwen2.5-1.5B"bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,  # Enable 4-bit quantization
    bnb_4bit_quant_type="nf4",  # Normal Float 4-bit
    bnb_4bit_compute_dtype=torch.bfloat16,  # Use bfloat16 for computation
    bnb_4bit_use_double_quant=True,  # Nested quantization for memory savings
)
```

- Load the model and tokenizer

```
# Load model
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    quantization_config=bnb_config,
    device_map="auto",
    trust_remote_code=True
)
```

- Display model device

```
# Check device of model parameters
for name, param in model.named_parameters():
    print(f"Parameter {name} is on device: {param.device}")
    break  # Print only the first parameter for brevity

# Alternatively, check a specific module
print(f"Model's first layer device: {next(model.parameters()).device}")
```

- Load the tokenizer

```
# Load tokenizer
tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token  # Set padding token
tokenizer.padding_side = "right"  # Pad on the right for causal LM

# Prepare model for k-bit training
model = prepare_model_for_kbit_training(model)
```

- Configure SRT configuration

```
from trl import SFTConfig

# Update training_args
training_args = SFTConfig(
    # Your existing arguments, e.g., output_dir, learning_rate, etc.
    output_dir="./sftqwen_finetuned_model_1-5BHS",
    num_train_epochs=1,
    per_device_train_batch_size=2,
    per_device_eval_batch_size=2,
    gradient_accumulation_steps=5,
    learning_rate=2e-4,
    warmup_steps=10,
    max_grad_norm=0.3,
    weight_decay=0.001,
    eval_strategy="steps",  # Updated from evaluation_strategy
    eval_steps=10,
    save_strategy="steps",
    save_steps=10,
    save_total_limit=2,
    logging_steps=5,
    fp16=False,
    bf16=True,
    optim="paged_adamw_8bit",
    report_to="none",
    load_best_model_at_end=True,
    metric_for_best_model="loss",
    completion_only_loss=False,  # Disable completion-only loss
    # ... other arguments ...
)
```

- function to format for Qwen 2.5 - 1.5B fine-tuning

```
def messages_to_string(messages, eos_token="</s>"):
    conversation = ""
    for msg in messages:
        role = msg["role"]
        content = msg["content"]
        conversation += f"<|{role}|>\n{content}\n"
    if not conversation.endswith(eos_token):
        conversation += eos_token
    return conversation
```

```
import json
converted_data = []

def formatting_func(example):
    try:
        prompt = example['prompt']
        response = example['completion']
        conversation = ""

        # Only process if both prompt and response are non-empty
        if prompt and response:
            messages = [
                {"role": "user", "content": prompt},
                {"role": "assistant", "content": response}
            ]
            converted_data.append({"messages": messages})
            example["messages"] = messages_to_string(messages)
            
        else:
            print(f"Skipped line : Missing prompt or completion.")
    except json.JSONDecodeError as e:
        print(f"Error parsing line : {e}")
    if "messages" in example and not example["messages"].endswith(tokenizer.eos_token):
                example["messages"] = example["messages"] + tokenizer.eos_token
    return example["messages"]
```

- rename the dataset

```
# Rename 'response' to 'completion'
train_dataset = train_dataset.rename_column("response", "completion")
val_dataset = val_dataset.rename_column("response", "completion")
```

- Configure the trainer

```
trainer = SFTTrainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=val_dataset,
    peft_config=lora_config,
    formatting_func=formatting_func,  # Use formatting function
    # max_seq_length=256,
    # packing=True,
)
```

- load the ipywidgets extension

```
import ipywidgets as widgets
from IPython.display import display
```

- Start the training process

```
# Step 7: Train and Validate
print("Starting fine-tuning...")
trainer.train()
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/amlqwen2-5-1B-1.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/amlqwen2-5-1B-2.jpg 'RagChat')

- Status
- Took 10h and 13 minutes to complete the training on a 2xA100 GPU cluster.
- SKU: Standard_NC24ads_A100_v4 (24 cores, 220 GB RAM, 64 GB disk)

```
# Save the final model
trainer.save_model("./sftqwen_finetuned_model_1-5BHS/final")
tokenizer.save_pretrained("./sftqwen_finetuned_model_1-5BHS/final")
print("Model saved.")
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/amlqwen2-5-1B-2.jpg 'RagChat')

- Save the model
- Evaluate the model

```
# Step 8: Evaluate the Model
eval_results = trainer.evaluate()
print(f"Validation Loss: {eval_results['eval_loss']}")
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/amlqwen2-5-1B-4.jpg 'RagChat')

- Upload the model to Hugging Face

```
model.push_to_hub("Balab2021/1B_finetuned_llama3.2_HS")
tokenizer.push_to_hub("Balab2021/1B_finetuned_llama3.2_HS")
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/amlqwen2-5-1B-5.jpg 'RagChat')

## Conclusion

- Took 6 hours and 22 minutes to fine-tune the model.
- Make sure we format the model input based on model input signature.
- Using SKU as Standard_NC24ads_A100_v4 (24 cores, 220 GB RAM, 64 GB disk) is recommended.
- Data set is about 37,120 rows of 170MB data size for training
- Validation data set is about 5MB.