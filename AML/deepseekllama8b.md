# Azure Machine learning inferencing with Deep Seek R1 LLama 8B

## Introduction

- This notebook demonstrates how to deploy a llama3.2 1B model on Azure Machine Learning (AML) using jupyter notebook.
- Huggingface llama3.2 1B model
- Testing on smaller footprint.

## Prerequisites

- Azure subscription
- Azure Machine Learning workspace
- GPU compute cluster
- Standard_NC64as_T4_v3 (64 cores, 440 GB RAM, 2816 GB disk)
- python 3.12
- Huggingface - deepseek-ai/DeepSeek-R1-Distill-Llama-8B - https://huggingface.co/deepseek-ai/DeepSeek-R1-Distill-Llama-8B
- Just testing text generation for now.
- python 3.12 virtual environment with latest packages installed.

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
model_id = "deepseek-ai/DeepSeek-R1-Distill-Llama-8B"
```

- Load the model

```
pipe = pipeline(
    "text-generation", 
    model=model_id, 
    torch_dtype=torch.bfloat16, 
    device_map="auto",
    max_new_tokens = 500,
    #temperature=0.0,
)
)
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/deepseekllama8b-2.jpg 'RagChat')

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/deepseekllama8b-3.jpg 'RagChat')

- Test the model

```
result = pipe("write a code to build a reinforcement learning for neural nets")
```

- Print the response

```
print(response[0]['generated_text'])
```

- output

```
write a code to build a reinforcement learning for neural nets

Okay, so I need to write a code to build a reinforcement learning for neural networks. Hmm, where do I start? I'm a bit new to this, but I remember that reinforcement learning involves agents learning to make decisions by interacting with an environment. The agent gets rewards or penalties based on its actions, and over time, it learns a policy that maximizes the cumulative reward.

First, I think I need to set up the environment. Maybe a simple grid world where the agent can move around and collect rewards. I remember something about the classic grid world problem where the agent is at the bottom-left corner and needs to reach the top-right corner without falling into holes. That could be a good starting point.

So, I'll define the grid dimensions. Let's say the grid is 4x4. The agent starts at position (0,0). The goal is at (3,3). There are some holes, like (1,1) and (2,2), which are dangerous. The agent can move up, down, left, or right, but not into the holes or out of the grid.

Next, I need to model the agent's actions. The agent can choose to move in four directions. But sometimes it might not move, maybe to think or explore. So, I'll have a policy that decides the action based on the current state.

I think I'll use a DQN (Deep Q-Network) approach because it's a popular method. DQN combines Q-learning with deep neural networks to handle high-dimensional state spaces. So, I'll need a neural network to approximate the Q-values.

The neural network will have input states, hidden layers, and output Q-values. Let's say the input is the current position of the agent, which is two integers (x, y). The hidden layers can be two layers with 64 and 64 neurons each, using ReLU activation. The output will be the Q-values for each possible action.

I'll also need a target network to compute the target Q-values for training. The target network will have the same structure as the main network but will be updated with the target values.

For the Q-learning part, I need to calculate the reward and the next state. The reward is 1 if the agent moves into the goal, -1 if it falls into a hole, and 0 otherwise. The next state is the new position after moving.
```

![info](https://github.com/balakreshnan/Samples2025/blob/main/AML/images/deepseekllama8b-5.jpg 'RagChat')

- Done
- Next will be reasoning testing.