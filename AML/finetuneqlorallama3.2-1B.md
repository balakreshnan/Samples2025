# Azure Machine Learning (AML) - Fine-tune QLoRA for Llama 3.2 1B

## Overview

- Fine tune a small language model using Azure Machine Learning (AML) and QLoRA.
- SKU used: Standard_NC24ads_A100_v4 (24 cores, 220 GB RAM, 64 GB disk).
- Using notebook to run the training job.
- I am using compute instance with GPU for this demo.
- Create a new Python 3.12 environment
- Fine tuning with LLama 3.2 1B model using QLoRA.

## Pre-requisites

- Azure subscription
- Azure Machine Learning workspace
- Azure Machine Learning compute instance with GPU
- SKU used: Standard_NC24ads_A100_v4 (24 cores, 220 GB RAM, 64 GB disk)
- Create a new Python 3.12 environment

## Steps

- Createa a python 3.12 environment in Azure Machine Learning workspace

```
conda create --name py312llm
conda activate py312llm
conda install python==3.12
conda install ipykernel
python -m ipykernel install --user --name py312llm --display-name "Python (py312llm)"
```

```
pip install transformers peft bitsandbytes datasets accelerate torch
pip install trl  # For Supervised Fine-Tuning (SFT) utilities
pip install huggingface_hub  # For uploading models to Hugging Face Hub
```

```
%pip install ipywidgets
```

- Now import the required libraries

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig, TrainingArguments
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from datasets import Dataset
from trl import SFTTrainer
from huggingface_hub import login
import pandas as pd
import os
```

- Clear the cache from Azure machine learning compute instance

```
!ls ~/.cache/huggingface/hub
!rm -rf ~/.cache/huggingface/hub
!ls ~/.cache/huggingface/hub
```

- Login to Hugging Face Hub

```
from huggingface_hub import notebook_login

notebook_login()
```

- Sample data set to be used for fine tuning

```
{"question": "What is the full name of the player from PBKS?", "answer": "The full name of the player from PBKS is Mayank Anurag Agarwal."}
{"question": "How many matches has Mayank Agarwal played?", "answer": "Mayank Agarwal has played 100 matches."}
{"question": "What is Mayank Agarwal's batting style?", "answer": "Mayank Agarwal's batting style is right-handed."}
{"question": "How many centuries has Mayank Agarwal scored in his career?", "answer": "Mayank Agarwal has scored 1 century in his career."}
{"question": "What is the highest score Mayank Agarwal has achieved in an innings?", "answer": "The highest score Mayank Agarwal has achieved in an innings is 106 v RR."}
{"question": "How many half-centuries has Mayank Agarwal scored?", "answer": "Mayank Agarwal has scored 11 half-centuries."}
{"question": "What is Mayank Agarwal's batting average?", "answer": "Mayank Agarwal's batting average is 23.41."}
{"question": "How many runs has Mayank Agarwal scored in his career?", "answer": "Mayank Agarwal has scored 2131 runs in his career."}
{"question": "How many catches has Mayank Agarwal taken?", "answer": "Mayank Agarwal has taken 40 catches."}
{"question": "What is Mayank Agarwal's strike rate?", "answer": "Mayank Agarwal's strike rate is 135.47."}
```

- once logged in now time to load jsonl data set

```
import pandas as pd
from datasets import Dataset
import json

# Step 1: Load the JSONL file line by line as dicts
#with open("crickdata.jsonl", "r") as f:
#    data = [json.loads(line) for line in f]  # Now this is a list of dicts

data = []
with open("crickdata.jsonl", "r") as f:
    for line in f:
        line = line.strip()
        if line:  # Skip empty lines
            try:
                record = json.loads(line)
                if isinstance(record, dict):
                    data.append(record)
                else:
                    print("Skipped non-dict JSON:", record)
            except json.JSONDecodeError as e:
                print("JSON decode error:", e, "Line:", line)

# Step 2: Convert to pandas DataFrame
df = pd.DataFrame(data)

# Step 3: Convert to Hugging Face Dataset
dataset = Dataset.from_pandas(df)

# Optional: View first row
print(dataset[0])
```

- print the length of the data set

```
print(f"Number of records: {len(dataset)}")
print(f"Features: {dataset.features}")
```

- Create train and validation data set

```
# Split into train (80%) and validation (20%)
dataset = dataset.train_test_split(test_size=0.2, seed=42)
train_dataset = dataset["train"]
val_dataset = dataset["test"]

# Format dataset for instruction tuning
def format_instruction(example):
    return f"### Instruction:\n{example['question']}\n### Response:\n{example['answer']}"

train_dataset = train_dataset.map(lambda x: {"text": format_instruction(x)})
val_dataset = val_dataset.map(lambda x: {"text": format_instruction(x)})
```

- Now load llama 3.2 1B instruct model

```
model_name = "meta-llama/Llama-3.2-1B-Instruct"
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,  # Enable 4-bit quantization
    bnb_4bit_quant_type="nf4",  # Normal Float 4-bit
    bnb_4bit_compute_dtype=torch.bfloat16,  # Use bfloat16 for computation
    bnb_4bit_use_double_quant=True,  # Nested quantization for memory savings
)
```

- Load the model

```
# Load model
try:
    model = AutoModelForCausalLM.from_pretrained(
        model_name,
        quantization_config=bnb_config,
        device_map="auto",
        trust_remote_code=True
    )
except Exception as e:
    print(f"Error loading {model_name}: {e}")
    model_name = "mistralai/Mixtral-7B-v0.1"  # Fallback model
    model = AutoModelForCausalLM.from_pretrained(
        model_name,
        quantization_config=bnb_config,
        device_map="auto",
        trust_remote_code=True
    )
```

- now tokernizer and LoRa configuration

```
# Load tokenizer
tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token  # Set padding token
tokenizer.padding_side = "right"  # Pad on the right for causal LM

# Prepare model for k-bit training
model = prepare_model_for_kbit_training(model)

# Step 4: Configure LoRA
lora_config = LoraConfig(
    r=16,  # LoRA rank
    lora_alpha=32,  # Scaling factor
    target_modules=["q_proj", "v_proj", "k_proj", "o_proj"],  # Target attention layers
    lora_dropout=0.05,  # Dropout for regularization
    bias="none",
    task_type="CAUSAL_LM"
)
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama3-2-1B-1.jpg 'RagChat')

- Delete the old fine tuned model if it exists

```
!rm -rf ./qlora_finetuned_model
```

- Create the model with LoRa configuration

```
# Apply LoRA
model = get_peft_model(model, lora_config)
model.print_trainable_parameters()  # Check trainable parameters (~1% of total)

# Step 5: Set Up Training Arguments
training_args = TrainingArguments(
    output_dir="./qlora_finetuned_model",
    num_train_epochs=50,
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
)
```

- initialize the trainer

```
# Step 6: Initialize SFTTrainer
trainer = SFTTrainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=val_dataset,
    peft_config=lora_config,  # Pass LoRA config explicitly
    # dataset_text_field="text",
    # max_seq_length=256,
    # packing=True,
)
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama3-2-1B-2.jpg 'RagChat')

- Start the training

```
# Step 7: Train and Validate
print("Starting fine-tuning...")
trainer.train()
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama3-2-1B-6.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama3-2-1B-7.jpg 'RagChat')

- Save the model

```
# Save the final model
trainer.save_model("./qlora_finetuned_model/final")
tokenizer.save_pretrained("./qlora_finetuned_model/final")
print("Model saved.")
```

- Evaluate the model

```
# Step 8: Evaluate the Model
eval_results = trainer.evaluate()
print(f"Validation Loss: {eval_results['eval_loss']}")
```

- Validate the model

```
# Step 9: Inference with Fine-Tuned Model
from peft import PeftModel

# Load fine-tuned model
base_model = AutoModelForCausalLM.from_pretrained(
    model_name,
    quantization_config=bnb_config,
    device_map="auto"
)
fine_tuned_model = PeftModel.from_pretrained(base_model, "./qlora_finetuned_model/final")
tokenizer = AutoTokenizer.from_pretrained("./qlora_finetuned_model/final")

# Test inference
prompt = "How many sixes has Liam Livingstone hit?"
inputs = tokenizer(f"### Instruction:\n{prompt}\n### Response:", return_tensors="pt").to("cuda")
outputs = fine_tuned_model.generate(**inputs, max_length=50)
print("Inference Output:")
print(tokenizer.decode(outputs[0], skip_special_tokens=True))
```

- Output

```
Setting `pad_token_id` to `eos_token_id`:128001 for open-end generation.
Inference Output:
### Instruction:
How many sixes has Liam Livingstone hit?
### Response: 
Liam Livingstone has hit six sixes in his career.
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama3-2-1B-8.jpg 'RagChat')

- I was only testing process of fine tuning a small model using QLoRA.
- This is not a production ready model.
- Need more datasets and training time to make it production ready.
- Hugging face libraries makes it easier to fine tune a model.
- Done for now.