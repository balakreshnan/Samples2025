# Fine tuning Azure OpenAI models with helpsteer using gpt-4.1-nano - Azure AI Foundry

## Introduction

- Take a custom dataset and fine-tune a model using gpt-4.1-nano.
- For this example, we will use the HelpSteer dataset, which is a collection of helpdesk conversations.
- The goal is to fine-tune the model to better understand and respond to helpdesk queries.
- Using Fine tuning as a service (FTaaS) with Azure OpenAI.
- This guide assumes you have a basic understanding of Python and Azure OpenAI.
- Need to format the dataset in a specific way for fine-tuning.
- gpt-4.1-nano requires chat conversation format for fine-tuning.
- HelpSteer dataset is has prompt and response pairs with other columns that are not needed for fine-tuning.
- I am testing in EAST US 2 region.
- Using Azure AI Foundry project.

## Code to convert HelpSteer dataset to fine-tuning format

- Here is where i download the HelpSteer dataset and convert it to the required format for fine-tuning.
- [helpsteer](https://huggingface.co/datasets/nvidia/HelpSteer)
- data set in in JSONL format
- The code below will convert the dataset to the required format for fine-tuning with gpt-4.1-nano.

```
import json

# Input and output file paths
#input_path = 'finetunedata/train.jsonl'
#output_path = 'finetunedata/converted_train.jsonl'

input_path = 'finetunedata/val.jsonl'
output_path = 'finetunedata/converted_val.jsonl'

# System message to use for all entries
system_message = {"role": "system", "content": "I am AI assistant to help with your queries."}

with open(input_path, 'r', encoding='utf-8') as infile, open(output_path, 'w', encoding='utf-8') as outfile:
    for line in infile:
        try:
            data = json.loads(line)
            prompt = data.get('prompt', '').strip()
            response = data.get('response', '').strip()

            converted = {
                "messages": [
                    system_message,
                    {"role": "user", "content": prompt},
                    {"role": "assistant", "content": response}
                ]
            }

            # Write each converted object as a new line in output file
            outfile.write(json.dumps(converted, ensure_ascii=False) + '\n')
        except json.JSONDecodeError as e:
            print(f"Skipping invalid JSON line: {e}")
```

- There are two files, one for training and one for validation.
- Switch training and validation files by changing the input and output file paths.
- Then creaet the 2 new files in the finetunedata folder.

## Azure AI Foundry fine tuning

- Now that we have the dataset in the required format, we can use Azure AI Foundry to fine-tune the model.
- Go to your Azure AI Foundry project.
- Click on the "Fine-tune" tab.
- Click Fine tune model.
- Select the model as gpt4.1-nano

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/gpt41nanoft-1.jpg 'RagChat')

- Next upload the training file from local computer.

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/gpt41nanoft-2.jpg 'RagChat')

- Next upload the validation file from local computer.

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/gpt41nanoft-3.jpg 'RagChat')

- Select Global as the region.
- provide a prefix for the fine-tuned model.
- i choose to leave the other options as default.
- Click Submit.

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/gpt41nanoft-4.jpg 'RagChat')

- Now time to wait for the fine-tuning to complete.
- Watch some movies or go for a walk.
- Or create epic AI software like i do.
- Once the fine-tuning is complete, you will see the new model in your Azure AI Foundry project.

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/gpt41nanoft-5.jpg 'RagChat')

- here is the metric output for the fine-tuned model.

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/gpt41nanoft-6.jpg 'RagChat')

- In my case it took about 7h 10m 56s.
- Training file is 37,120 rows of data.
- Training file size is 107MB
- Validation file is 5MB.
- Need to work evaluation of the model.
- Training tokens billed: 23,515,000

## Conclusion

- We have successfully fine-tuned a gpt-4.1-nano model using the HelpSteer dataset.
- This model can now be used to better understand and respond to helpdesk queries.
- You can use this model in your applications to provide better support and assistance to users.
- Can help with industry specific use cases. 