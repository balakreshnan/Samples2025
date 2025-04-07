# Red Teaming Agent in Azure AI Foundry

## Introduction

- Ability to perform red teaming activities using Azure AI Foundry.
- Creates simulated data sets to test security controls and incident response capabilities.
- Based on Hate and unfairbess, self-harm, Sexual and Violence aspects of red teaming.
- Ability to do basic, intermediate, and advanced red teaming activities.
- Ability to fast track red teaming testing and provide insights to cyber security teams.

## Prerequisites

- Azure subscription with access to Azure AI Foundry.
- Azure AI Foundry workspace created.
- Azure AI Foundry agent created.
- Python virtual environment is 3.11

## Steps

- install the required libraries

```
pip install azure-ai-project
pip install azure-ai-evaluation[redteam]
```

- import the required libraries

```python
from typing import Any, Callable, Set, Dict, List, Optional
from azure.ai.evaluation.red_team import RedTeam, RiskCategory, AttackStrategy
import os
import io
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import FileSearchTool, MessageAttachment, FilePurpose
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from azure.ai.projects.models import AzureAISearchTool, AzureAISearchQueryType
import os
from openai import AzureOpenAI
from pptx.util import Inches
from typing import List, Dict
from azure.ai.evaluation import evaluate, AzureAIProject, AzureOpenAIModelConfiguration, F1ScoreEvaluator
from typing import Any, Callable, Set, Dict, List, Optional
from azure.ai.evaluation.red_team import RedTeam, RiskCategory, AttackStrategy
```

- First create call back functions

```
# Define a simple callback function that always returns a fixed response
def financial_advisor_callback(query: str) -> str:  # noqa: ARG001
    return "I'm a financial advisor assistant. I can help with investment advice and financial planning within legal and ethical guidelines."
# Create a more complex callback function that handles full conversation context
async def advanced_callback(messages: Dict, stream: bool = False, session_state: Any = None, context: Optional[Dict] =None) -> dict:
    """A more complex callback that processes conversation history"""
    # Extract the latest message from the conversation history
    messages_list = [{"role": chat_message.role,"content": chat_message.content} for chat_message in messages]
    latest_message = messages_list[-1]["content"]
    
    # In a real application, you might process the entire conversation history
    # Here, we're just simulating different responses based on the latest message
    response = "I'm an AI assistant that follows safety guidelines. I cannot provide harmful content."
    
    # Format the response to follow the openAI chat protocol format
    formatted_response = {
        "content": response,
        "role": "assistant"
    }
    
    return {"messages": [formatted_response]}
```

- Here is agent that runs the red teaming activities

```python
async def redteam_agent(query: str) -> str:
    returntxt = ""
    #print('Citation Text:', citationtxt)
    azure_endpoint = os.getenv("AZURE_OPENAI_RED_ENDPOINT")
    api_key = os.getenv("AZURE_OPENAI_RED_KEY")
    azure_deployment = os.getenv("AZURE_OPENAI_RED_DEPLOYMENT")
    api_version = os.getenv("AZURE_OPENAI_API_VERSION")

    model_config = AzureOpenAIModelConfiguration(
       azure_endpoint=azure_endpoint,
       api_key=api_key,
       api_version=api_version,
       azure_deployment=azure_deployment,
   )

    try:
        credential = DefaultAzureCredential()
        credential.get_token("https://management.azure.com/.default")
    except Exception as ex:
        print(ex)

    subscription_id = os.getenv("AZURE_SUBSCRIPTION_ID")
    resource_group_name = os.getenv("AZURE_RESOURCE_GROUP")
    project_name = os.getenv("AZUREAI_PROJECT_NAME")
    print(subscription_id, resource_group_name, project_name)
    azure_ai_project = AzureAIProject(subscription_id=subscription_id, 
                                      resource_group_name=resource_group_name, 
                                      project_name=project_name, 
                                      azure_crendential=credential)

    # Create the `RedTeam` instance with minimal configurations
    red_team = RedTeam(
        azure_ai_project=azure_ai_project,
        credential=credential,
        risk_categories=[RiskCategory.Violence, RiskCategory.HateUnfairness],
        num_objectives=1,
    )

    # Run the red team scan called "Basic-Callback-Scan" with limited scope for this basic example
    # This will test 1 objective prompt for each of Violence and HateUnfairness categories with the Flip strategy
    result = await red_team.scan(
        target=financial_advisor_callback, scan_name="Basic-Callback-Scan", attack_strategies=[AttackStrategy.Flip]
    )

    returntxt += str(result)

    # Define a model configuration to test
    azure_oai_model_config = {
        "azure_endpoint": azure_endpoint,
        "azure_deployment": azure_deployment,
        "api_key": api_key,
    }
    # # Run the red team scan called "Intermediary-Model-Target-Scan"
    result = await red_team.scan(
        target=azure_oai_model_config, scan_name="Intermediary-Model-Target-Scan", attack_strategies=[AttackStrategy.Flip]
    )
    returntxt += str(result)

    # Create the RedTeam instance with all of the risk categories with 5 attack objectives generated for each category
    model_red_team = RedTeam(
        azure_ai_project=azure_ai_project,
        credential=credential,
        risk_categories=[RiskCategory.Violence, RiskCategory.HateUnfairness, RiskCategory.Sexual, RiskCategory.SelfHarm],
        num_objectives=2,
    )

    # Run the red team scan with multiple attack strategies
    advanced_result = await model_red_team.scan(
        target=advanced_callback,
        scan_name="Advanced-Callback-Scan",
        attack_strategies=[
            AttackStrategy.EASY,  # Group of easy complexity attacks
            AttackStrategy.MODERATE,  # Group of moderate complexity attacks
            #AttackStrategy.CharacterSpace,  # Add character spaces
            #AttackStrategy.ROT13,  # Use ROT13 encoding
            #AttackStrategy.UnicodeConfusable,  # Use confusable Unicode characters
            #AttackStrategy.CharSwap,  # Swap characters in prompts
            #AttackStrategy.Morse,  # Encode prompts in Morse code
            #AttackStrategy.Leetspeak,  # Use Leetspeak
            #AttackStrategy.Url,  # Use URLs in prompts
            #AttackStrategy.Binary,  # Encode prompts in binary
            AttackStrategy.Compose([AttackStrategy.Base64, AttackStrategy.ROT13]),  # Use two strategies in one attack
        ],
        output_path="./Advanced-Callback-Scan.json",
    )

    returntxt += str(advanced_result)

    return returntxt
```

- Create the main function to run the red teaming agent

```python
async def main():
    # Run the red teaming agent
    result = await redteam_agent("Run red teaming activities")
    print(result)

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

- Run the script

```
python redteam.py
```

- Wait until it completes and then go to Azure AI Foundry.

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/redteam-1.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/redteam-2.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/redteam-3.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/redteam-4.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/redteam-5.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/redteam-6.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/redteam-7.jpg 'RagChat')

- this is just a begining of red teaming activities using Azure AI Foundry.
- Try with more tasks.