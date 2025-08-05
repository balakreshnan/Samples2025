# OpenAI 20B Model Fine Tuning on Azure ML

## Pre-requisites

- Azure subscription
- Azure ML workspace
- Python 3.12 installed
- GPU SKU: Standard_NC96ads_A100_v4 (96 cores, 880 GB RAM, 256 GB disk)
- 4 GPU A100 in West us3 region
- Using model from Hugging Face: `openai/gpt-oss-20b`
- Thank you to open ai for providing the code: https://cookbook.openai.com/articles/gpt-oss/fine-tune-transfomers
- Just to show case how to fine tune the model on Azure ML

## Steps

- Create a new Azure ML Environment with Python 3.12

```
conda create --name py312
conda activate py312
conda install python==3.12
conda install ipykernel
python -m ipykernel install --user --name py312 --display-name "Python (py312)"
```

- Now install the necessary libraries

```
%pip install torch --index-url https://download.pytorch.org/whl/cu128
```

```
%pip install "trl>=0.20.0" "peft>=0.17.0" "transformers>=4.55.0" trackio
```

- to clear the cache

```
!rm -rf ~/.cache/huggingface/hub
```

- Login to Hugging Face

```
from huggingface_hub import notebook_login

notebook_login()
```

- Load the dataset

```
from datasets import load_dataset

dataset = load_dataset("HuggingFaceH4/Multilingual-Thinking", split="train")
dataset
```

```
dataset[0]
```

- Now lets load the model

```
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("openai/gpt-oss-20b")
```

- display the tokenizer

```
messages = dataset[0]["messages"]
conversation = tokenizer.apply_chat_template(messages, tokenize=False)
print(conversation)
```

- Configure the model

```
import torch
from transformers import AutoModelForCausalLM, Mxfp4Config

quantization_config = Mxfp4Config(dequantize=True)
model_kwargs = dict(
    attn_implementation="eager",
    torch_dtype=torch.bfloat16,
    quantization_config=quantization_config,
    use_cache=False,
    device_map="auto",
)

model = AutoModelForCausalLM.from_pretrained("openai/gpt-oss-20b", **model_kwargs)
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/openaiosmodel20bft-2.jpg 'RagChat')

- Test the model

```
messages = [
    {"role": "user", "content": "¿Cuál es el capital de Australia?"},
]

input_ids = tokenizer.apply_chat_template(
    messages,
    add_generation_prompt=True,
    return_tensors="pt",
).to(model.device)

output_ids = model.generate(input_ids, max_new_tokens=512)
response = tokenizer.batch_decode(output_ids)[0]
print(response)
```

- Left configure the peft

```
from peft import LoraConfig, get_peft_model

peft_config = LoraConfig(
    r=8,
    lora_alpha=16,
    target_modules="all-linear",
    target_parameters=[
        "7.mlp.experts.gate_up_proj",
        "7.mlp.experts.down_proj",
        "15.mlp.experts.gate_up_proj",
        "15.mlp.experts.down_proj",
        "23.mlp.experts.gate_up_proj",
        "23.mlp.experts.down_proj",
    ],
)
peft_model = get_peft_model(model, peft_config)
peft_model.print_trainable_parameters()
```

- Configure SFT config

```
from trl import SFTConfig

training_args = SFTConfig(
    learning_rate=2e-4,
    gradient_checkpointing=True,
    num_train_epochs=1,
    logging_steps=1,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,
    max_length=2048,
    warmup_ratio=0.03,
    lr_scheduler_type="cosine_with_min_lr",
    lr_scheduler_kwargs={"min_lr_rate": 0.1},
    output_dir="gpt-oss-20b-multilingual-reasoner",
    report_to="trackio",
    push_to_hub=True,
)
```

- Now train the model

```
from trl import SFTTrainer

trainer = SFTTrainer(
    model=peft_model,
    args=training_args,
    train_dataset=dataset,
    processing_class=tokenizer,
)
trainer.train()
```

- In progess you can see the training

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/openaiosmodel20bft-1.jpg 'RagChat')

- Once completed, you can see the training results

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/openaiosmodel20bft-3.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/openaiosmodel20bft-4.jpg 'RagChat')

- output

```
TrainOutput(global_step=63, training_loss=1.1387268096681624, metrics={'train_runtime': 1703.0347, 'train_samples_per_second': 0.587, 'train_steps_per_second': 0.037, 'total_flos': 1.9951547028611328e+17, 'train_loss': 1.1387268096681624})
```

- Save the model

```
trainer.save_model(training_args.output_dir)
trainer.push_to_hub(dataset_name="Balab2021/openai20bMultilingual-Thinking")
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/openaiosmodel20bft-5.jpg 'RagChat')

- Now inference testing
- test with sample input

```
from transformers import pipeline

question = "If you had a time machine, but could only go to the past or the future once and never return, which would you choose and why?"
generator = pipeline("text-generation", model="Balab2021/gpt-oss-20b-multilingual-reasoner", device="cuda")
output = generator([{"role": "user", "content": question}], max_new_tokens=128, return_full_text=False)[0]
print(output["generated_text"])
```

## Conclusion

- In this notebook we have seen how to fine tune the OpenAI 20B model on Azure ML using PEFT and SFTTrainer from TRL library.
- was able to fine tune with SKU: Standard_NC96ads_A100_v4 (96 cores, 880 GB RAM, 256 GB disk)
- Now to try with custom dataset and see the results.
- Inference testing looks good.
- Happy Learning :)