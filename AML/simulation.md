# Azure ai Evaluation SDK - Simulation Code

## Introduction

- Simualate data for the purpose of testing the model.
- Simualte data for fine tuning the model.
- Also learn about using the model to get text.

## Prerequisites

- Azure Subscription
- Azure AI foundation account
- Azure Open AI account
- Following tutorials from: https://github.com/MicrosoftDocs/azure-docs-sdk-python/blob/main/docs-ref-services/latest/ai-evaluation-readme.md

## Steps

- First import libraries

```
import os
import asyncio
from typing import Any, Dict, List, Optional
from azure.ai.evaluation.simulator import Simulator
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.ai.projects.models import AzureAISearchTool
# [START enable_tracing]
from openai import AzureOpenAI
from datetime import datetime
import streamlit as st
import datetime
from azure.ai.evaluation import BleuScoreEvaluator, GleuScoreEvaluator, RougeScoreEvaluator, MeteorScoreEvaluator, RougeType
from dotenv import load_dotenv

# Load .env file
load_dotenv()
```

- Now client meeting

```
client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT"), 
  api_key=os.getenv("AZURE_OPENAI_API_KEY"),  
  api_version="2024-10-21",
)

model_name = os.getenv("AZURE_OPENAI_DEPLOYMENT")
```

- Setup model config

```
model_config = {
    "azure_endpoint": os.environ.get("AZURE_OPENAI_ENDPOINT"),
    "api_key": os.environ.get("AZURE_OPENAI_API_KEY"),
    "azure_deployment": os.environ.get("AZURE_OPENAI_DEPLOYMENT"),
}
```

- Call the open ai model

```
def callopenai(query, messages_list, context):
    returntxt = ""

    response = client.chat.completions.create(
        model=os.getenv("AZURE_OPENAI_DEPLOYMENT"), #"gpt-4-turbo", # model = "deployment_name".
        messages=messages_list,
        temperature=0.0,
        top_p=0.0,
        seed=42,
        max_tokens=1000,
    )

    returntxt = response.choices[0].message.content
    return returntxt
```

- Create the call back

```
async def callback(
    messages: Dict[str, List[Dict]],
    stream: bool = False,
    session_state: Any = None,
    context: Optional[Dict[str, Any]] = None,
) -> dict:
    messages_list = messages["messages"]
    # Get the last message from the user
    latest_message = messages_list[-1]
    query = latest_message["content"]
    # Call your endpoint or AI application here
    # response should be a string
    response = callopenai(query, messages_list, context)
    formatted_response = {
        "content": response,
        "role": "assistant",
        "context": "",
    }
    messages["messages"].append(formatted_response)
    return {"messages": messages["messages"], "stream": stream, "session_state": session_state, "context": context}
```

- function to parse the response

```
# Function to extract the latest assistant message
def get_latest_assistant_message(conversations):
    for conversation in reversed(conversations):  # Start from the latest conversation
        for message in reversed(conversation["messages"]):  # Look at messages in reverse order
            if message["role"] == "assistant":
                return message["content"]
    return None  # Return None if no assistant message is found
```

- Create a simulator

```
simulator = Simulator(model_config=model_config)
```

- now call the main and run the function

```
if __name__ == "__main__":
    # asyncio.run(callback)
    custom_simulator = Simulator(model_config=model_config)
    outputs = asyncio.run(custom_simulator(
        target=callback,
        conversation_turns=[
            [
                "What should I know about the public gardens in the US?",
            ],
            [
                "How do I simulate data against LLMs",
            ],
        ],
        max_conversation_turns=2,
    ))
    # print(outputs)
    # Extract and display the latest assistant message
    latest_message = get_latest_assistant_message(outputs)
    if latest_message:
        print("Latest Assistant Message:")
        print(latest_message)
    else:
        print("No assistant messages found.")
    with open("simulator_output.jsonl", "w") as f:
        for output in outputs:
            f.write(output.to_eval_qr_json_lines())
```

- Run the code

```
python simulation.py
```

- here is the output

```
Latest Assistant Message:
Choosing the right prompts is crucial for effective data generation with large language models. Here are some best practices and examples to help you craft effective prompts:

### Best Practices for Crafting Prompts

1. **Clarity and Specificity**: Ensure your prompts are clear and specific. Ambiguous prompts can lead to unpredictable or irrelevant outputs. Specify the context, style, or format you expect in the response.

2. **Contextual Information**: Provide enough context to guide the model. This might include background information, examples, or specific instructions.

3. **Iterative Refinement**: Start with a basic prompt and refine it based on the outputs you receive. Adjust the wording, add constraints, or provide additional context as needed.

4. **Use Examples**: If possible, include examples in your prompt to demonstrate the desired output style or format. This can help the model understand your expectations better.

5. **Experiment with Variations**: Try different phrasings or structures to see which yields the best results. Sometimes small changes can significantly impact the quality of the output.

6. **Leverage Model Capabilities**: Understand the strengths and limitations of the model you're using. Tailor your prompts to leverage its strengths, such as language understanding or creative generation.

### Examples of Effective Prompts

1. **Creative Writing**:
   - **Prompt**: "Write a short story about a detective who solves a mystery using only clues found in a library. The story should be suspenseful and have a surprising twist at the end."
   - **Purpose**: This prompt provides a clear scenario, genre, and desired tone, guiding the model to generate a coherent and engaging story.

2. **Technical Explanation**:
   - **Prompt**: "Explain the concept of blockchain technology to a 10-year-old. Use simple language and analogies to make it easy to understand."
   - **Purpose**: This prompt specifies the audience and the level of complexity, helping the model tailor its response appropriately.

3. **Data Generation for Training**:
   - **Prompt**: "Generate a list of 10 customer service email responses for common issues such as late delivery, product malfunction, and billing errors. Each response should be polite, empathetic, and offer a solution."
   - **Purpose**: This prompt outlines the format, content, and tone, ensuring the generated data is useful for training customer service models.

4. **Question and Answer**:
   - **Prompt**: "Provide a detailed answer to the question: 'What are the main causes of climate change?' Include at least three causes and explain each one briefly."
   - **Purpose**: This prompt directs the model to focus on specific content and structure, resulting in a comprehensive and informative response.

By following these best practices and using these examples as a guide, you can craft prompts that effectively guide LLMs to generate high-quality and relevant data. 
```

- have fun with the code and enjoy the simulation.
- more to come

### sample output jsonl file

- simulator_output.jsonl

```
{"query": "What should I know about the public gardens in the US?", "response": "Public gardens in the United States are diverse and offer a wide range of experiences, from botanical gardens and arboretums to historical estates and urban parks. Here are some key points to know about them:\n\n1. **Diversity and Specialization**: Public gardens often specialize in certain types of plants or themes. For example, some focus on native plants, while others might have extensive collections of roses, orchids, or succulents. Many gardens also have themed sections, such as Japanese gardens, desert landscapes, or tropical conservatories.\n\n2. **Educational Opportunities**: Many public gardens offer educational programs, workshops, and tours. These can range from gardening classes and plant identification workshops to lectures on environmental conservation and sustainability.\n\n3. **Conservation and Research**: Public gardens often play a crucial role in plant conservation and research. They may participate in efforts to preserve endangered plant species, conduct scientific research, and maintain seed banks.\n\n4. **Cultural and Historical Significance**: Some gardens are part of historical estates or have significant cultural heritage. They may include historic buildings, sculptures, and art installations, providing a glimpse into the past and the cultural context of the area.\n\n5. **Community Engagement**: Public gardens frequently serve as community hubs, hosting events such as plant sales, art exhibits, concerts, and seasonal festivals. They provide a space for relaxation, recreation, and social gatherings.\n\n6. **Accessibility and Amenities**: Most public gardens are designed to be accessible to a wide range of visitors, including those with mobility challenges. They often have amenities such as visitor centers, gift shops, cafes, and picnic areas.\n\n7. **Membership and Support**: Many public gardens offer membership programs that provide benefits like free admission, discounts, and access to special events. These memberships help support the garden's operations and initiatives.\n\n8. **Notable Public Gardens**: Some of the most famous public gardens in the U.S. include the New York Botanical Garden, Longwood Gardens in Pennsylvania, the United States Botanic Garden in Washington, D.C., and the Desert Botanical Garden in Arizona.\n\n9. **Seasonal Changes**: Public gardens offer different experiences throughout the year, with seasonal blooms, changing foliage, and special events. Visiting at different times can provide a new perspective and appreciation for the garden's offerings.\n\n10. **Environmental Impact**: Many public gardens are involved in promoting sustainable practices, such as water conservation, organic gardening, and habitat restoration. They often serve as models for environmentally friendly landscaping.\n\nVisiting public gardens can be a rewarding experience, offering beauty, education, and a connection to nature. Whether you're a plant enthusiast, a casual visitor, or someone looking for a peaceful retreat, there's likely a public garden that will meet your interests."}
{"query": "That sounds fascinating! Could you recommend some public gardens that are particularly known for their educational programs or conservation efforts? I'm interested in learning more about how they contribute to plant conservation and research.", "response": "Certainly! Many public gardens in the United States are renowned for their educational programs and conservation efforts. Here are a few notable ones:\n\n1. **Missouri Botanical Garden (St. Louis, Missouri)**: Known for its extensive research and conservation programs, the Missouri Botanical Garden is a leader in plant science. It offers a wide range of educational programs for all ages and is involved in global plant conservation efforts. The garden's research division, the William L. Brown Center, focuses on ethnobotany and the sustainable use of plant resources.\n\n2. **Chicago Botanic Garden (Glencoe, Illinois)**: This garden is heavily involved in plant conservation and research, particularly through its Plant Conservation Science Center. It offers numerous educational programs, including a graduate program in plant biology and conservation in partnership with Northwestern University. The garden's initiatives include habitat restoration and the conservation of native plant species.\n\n3. **New York Botanical Garden (Bronx, New York)**: The New York Botanical Garden is a major center for botanical research and education. It offers a wide array of educational programs, from children's workshops to professional training for botanists. The garden's Institute of Economic Botany focuses on the relationship between plants and people, and its herbarium is one of the largest in the world.\n\n4. **Fairchild Tropical Botanic Garden (Coral Gables, Florida)**: Specializing in tropical plants, Fairchild is dedicated to the conservation of tropical plant diversity. It offers educational programs for students and adults, including the Fairchild Challenge, an environmental education program for schools. The garden is also involved in international conservation projects and has a strong focus on orchid conservation.\n\n5. **Desert Botanical Garden (Phoenix, Arizona)**: This garden is dedicated to the conservation of desert plants and ecosystems. It offers a variety of educational programs and workshops focused on desert ecology and sustainable gardening practices. The garden's research efforts include the study of desert plant species and their adaptation to climate change.\n\n6. **Huntington Library, Art Museum, and Botanical Gardens (San Marino, California)**: The Huntington is known for its extensive botanical collections and research programs. It offers educational programs that cover a wide range of topics, from plant science to art and history. The garden's research focuses on plant conservation, particularly in relation to climate change and habitat loss.\n\n7. **Arnold Arboretum of Harvard University (Boston, Massachusetts)**: As a research institution, the Arnold Arboretum is deeply involved in the study of plant biology and conservation. It offers educational programs for the public and supports scientific research through its living collections and herbarium. The arboretum is a key player in the conservation of temperate woody plants.\n\nThese gardens not only provide beautiful landscapes to explore but also serve as important centers for education and conservation, contributing significantly to the understanding and preservation of plant biodiversity."}
{"query": "How do I simulate data against LLMs", "response": "Simulating data against large language models (LLMs) can be useful for testing, training, or evaluating the models. Here are some steps and considerations for simulating data:\n\n1. **Define Objectives**: Clearly define what you want to achieve with the simulation. Are you testing the model's ability to understand context, generate creative content, or perform specific tasks?\n\n2. **Select a Domain**: Choose the domain or topic for the data simulation. This could be general knowledge, specific industries, creative writing, etc.\n\n3. **Data Generation Tools**: Use tools and libraries that can help generate synthetic data. Some popular ones include:\n   - **GPT-3/4**: Use OpenAI's API to generate text based on prompts.\n   - **Hugging Face Transformers**: Utilize pre-trained models to generate text.\n   - **Data Augmentation Libraries**: Libraries like TextAttack or NLPAug can help create variations of existing data.\n\n4. **Create Prompts**: Design prompts that will guide the LLM to generate the desired type of data. Prompts should be clear and specific to elicit the best responses from the model.\n\n5. **Iterative Testing**: Generate data iteratively, refining prompts and parameters based on the output quality. This may involve adjusting temperature, max tokens, and other model parameters.\n\n6. **Evaluation**: Assess the generated data for quality, relevance, and diversity. You may need to manually review samples or use automated metrics like BLEU, ROUGE, or perplexity.\n\n7. **Bias and Fairness**: Be aware of potential biases in the generated data. Ensure that the data is fair and representative of the intended use case.\n\n8. **Feedback Loop**: Use feedback from evaluations to improve prompts and data generation strategies. This may involve retraining or fine-tuning the model if necessary.\n\n9. **Ethical Considerations**: Ensure that the data simulation respects privacy and ethical guidelines, especially if using real-world data as a basis for simulation.\n\n10. **Documentation**: Keep detailed records of the data generation process, including prompts, parameters, and evaluation criteria, to ensure reproducibility and transparency.\n\nBy following these steps, you can effectively simulate data against LLMs for various applications, from testing and evaluation to training and development."}
{"query": "Thank you for the detailed steps! I have a few follow-up questions. First, could you explain more about how to choose the right prompts for data generation? Are there any best practices or examples you could share?", "response": "Choosing the right prompts is crucial for effective data generation with large language models. Here are some best practices and examples to help you craft effective prompts:\n\n### Best Practices for Crafting Prompts\n\n1. **Clarity and Specificity**: Ensure your prompts are clear and specific. Ambiguous prompts can lead to unpredictable or irrelevant outputs. Specify the context, style, or format you expect in the response.\n\n2. **Contextual Information**: Provide enough context to guide the model. This might include background information, examples, or specific instructions.\n\n3. **Iterative Refinement**: Start with a basic prompt and refine it based on the outputs you receive. Adjust the wording, add constraints, or provide additional context as needed.\n\n4. **Use Examples**: If possible, include examples in your prompt to demonstrate the desired output style or format. This can help the model understand your expectations better.\n\n5. **Experiment with Variations**: Try different phrasings or structures to see which yields the best results. Sometimes small changes can significantly impact the quality of the output.\n\n6. **Leverage Model Capabilities**: Understand the strengths and limitations of the model you're using. Tailor your prompts to leverage its strengths, such as language understanding or creative generation.\n\n### Examples of Effective Prompts\n\n1. **Creative Writing**:\n   - **Prompt**: \"Write a short story about a detective who solves a mystery using only clues found in a library. The story should be suspenseful and have a surprising twist at the end.\"\n   - **Purpose**: This prompt provides a clear scenario, genre, and desired tone, guiding the model to generate a coherent and engaging story.\n\n2. **Technical Explanation**:\n   - **Prompt**: \"Explain the concept of blockchain technology to a 10-year-old. Use simple language and analogies to make it easy to understand.\"\n   - **Purpose**: This prompt specifies the audience and the level of complexity, helping the model tailor its response appropriately.\n\n3. **Data Generation for Training**:\n   - **Prompt**: \"Generate a list of 10 customer service email responses for common issues such as late delivery, product malfunction, and billing errors. Each response should be polite, empathetic, and offer a solution.\"\n   - **Purpose**: This prompt outlines the format, content, and tone, ensuring the generated data is useful for training customer service models.\n\n4. **Question and Answer**:\n   - **Prompt**: \"Provide a detailed answer to the question: 'What are the main causes of climate change?' Include at least three causes and explain each one briefly.\"\n   - **Purpose**: This prompt directs the model to focus on specific content and structure, resulting in a comprehensive and informative response.\n\nBy following these best practices and using these examples as a guide, you can craft prompts that effectively guide LLMs to generate high-quality and relevant data."}
```

- Enjoy the simulation