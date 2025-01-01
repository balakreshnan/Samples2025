# Azure Machine learning inferecing with llama3.2 1B using CPU

## Introduction

- This notebook demonstrates how to deploy a llama3.2 1B model on Azure Machine Learning (AML) using jupyter notebook.
- Huggingface llama3.2 1B model
- Testing on smaller footprint.

## Prerequisites

- Azure subscription
- Azure Machine Learning workspace
- CPU compute cluster
- Standard_D13_v2 (8 cores, 56 GB RAM, 400 GB disk)
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

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama32-1bcpu-1.jpg 'RagChat')

- Test the model

```
response = pipe("Explain Quantum computing with details", temperature=0.7, max_length=500)
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama32-1bcpu-2.jpg 'RagChat')

- Print the response

```
print(response[0]['generated_text'])
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/llama32-1bcpu-3.jpg 'RagChat')

- output

```
Explain Quantum computing with details
Quantum computing is a computer science that uses quantum bits, or qubits, to perform calculations. The qubits are like the bits of a computer, but they can hold more than one value at once. Quantum computing can be used to solve problems that are difficult or impossible to solve with traditional computers.
What is Quantum computing?
Quantum computing is a computer science that uses quantum bits, or qubits, to perform calculations. The qubits are like the bits of a computer, but they can hold more than one value at once. Quantum computing can be used to solve problems that are difficult or impossible to solve with traditional computers.
Why Quantum computing is important?
Quantum computing is important because it can be used to solve problems that are difficult or impossible to solve with traditional computers. Quantum computing can be used to solve problems that are difficult or impossible to solve with traditional computers. Quantum computing can be used to solve problems that are difficult or impossible to solve with traditional computers.
Quantum computing is a computer science that uses quantum bits, or qubits, to perform calculations. The qubits are like the bits of a computer, but they can hold more than one value at once. Quantum computing can be used to solve problems that are difficult or impossible to solve with traditional computers.
How Quantum computing works?
Quantum computing works by using qubits to perform calculations. The qubits are like the bits of a computer, but they can hold more than one value at once. Quantum computing can be used to solve problems that are difficult or impossible to solve with traditional computers.
What are the advantages of Quantum computing?
Quantum computing has many advantages over traditional computers. One advantage is that it can solve problems that are difficult or impossible to solve with traditional computers. Another advantage is that it can be used to solve problems that require a lot of memory, such as image processing or data compression.
What are the disadvantages of Quantum computing?
Quantum computing has some disadvantages. One disadvantage is that it is not as powerful as traditional computers. Another disadvantage is that it can be expensive to build and maintain. Finally, quantum computing is still in its early stages of development, so there are many questions about its long-term potential.
Quantum computing is a computer science that uses quantum bits, or qubits, to perform calculations. The qubits are like the bits of a computer, but they can hold more than one value at once. Quantum computing can be used to solve problems
```

## Conclusion

- We have successfully deployed llama3.2 1B model on Azure Machine Learning (AML) using jupyter notebook.
- This can be applied to local machine with smaller footprint.