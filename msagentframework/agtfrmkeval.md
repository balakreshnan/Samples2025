# Microsoft Agent Framework Evaluation using Azure AI Foundry Services - Realtime Example

## Overview

- This document provides an evaluation of the Microsoft Agent Framework (MAF) within the context of Azure AI Foundry services.
- The evaluation focuses on the capabilities, integration, and performance of MAF when orchestrating AI agents for various tasks.
- Ability to validate the agents quality and performance using Azure AI Foundry services.
- Evaluate as the agents are consumed by the applications on every request.

## Prerequisites

- Access to Azure AI Foundry services.
- Familiarity with Microsoft Agent Framework concepts and architecture.
- Basic understanding of AI agent orchestration and workflow management.
- Azure Subscription
- Azure Open AI Models
- Microsoft Agent Framework
- Azure AI Foundry Evaluation
- Make sure environment is set up with necessary permissions and configurations.

```
AZURE_OPENAI_ENDPOINT=<your-azure-openai-endpoint>
AZURE_OPENAI_KEY=<your-azure-openai-key>
AZURE_OPENAI_API_VERSION="2025-01-01-preview"
AZURE_OPENAI_DEPLOYMENT="gpt-4.1"
AZURE_AI_PROJECT=""
APPLICATIONINSIGHTS_CONNECTION_STRING="<your-application-insights-connection-string>"
ENABLE_OTEL=true
```

- Create a python evvironment and install the required packages.

```
python -m venv venv
pip install -r requirements.txt
```

## Code

- Now we are going to create a simple agent using the Microsoft Agent Framework and evaluate it using Azure AI Foundry services.
- Once we execute the agent, then we will take the input and output and validate it using Azure AI Foundry services.

```
import asyncio
import os
from random import randint
from typing import Annotated
from urllib import response

from agent_framework import ChatAgent
from agent_framework.azure import AzureAIAgentClient
from azure.ai.projects.aio import AIProjectClient
from azure.identity.aio import AzureCliCredential
from pydantic import Field
import os
from azure.ai.evaluation import ToolCallAccuracyEvaluator, AzureOpenAIModelConfiguration
from azure.ai.evaluation import IntentResolutionEvaluator, TaskAdherenceEvaluator, ResponseCompletenessEvaluator

from pprint import pprint

from dotenv import load_dotenv

# Load environment variables
load_dotenv()

def get_weather(
    location: Annotated[str, Field(description="The location to get the weather for.")],
) -> str:
    """Get the weather for a given location."""
    conditions = ["sunny", "cloudy", "rainy", "stormy"]
    return f"The weather in {location} is {conditions[randint(0, 3)]} with a high of {randint(10, 30)}°C."

model_config = AzureOpenAIModelConfiguration(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_key=os.environ["AZURE_OPENAI_KEY"],
    api_version=os.environ["AZURE_OPENAI_API_VERSION"],
    azure_deployment=os.environ["AZURE_OPENAI_DEPLOYMENT"],
)


tool_call_accuracy = ToolCallAccuracyEvaluator(model_config=model_config)

async def main() -> None:
    print("=== Azure AI Chat Client with Existing Agent ===")

    # Create the client
    async with (
        AzureCliCredential() as credential,
        AIProjectClient(endpoint=os.environ["AZURE_AI_PROJECT_ENDPOINT"], credential=credential) as client,
    ):
        # Create an agent that will persist
        created_agent = await client.agents.create_agent(
            model=os.environ["AZURE_AI_MODEL_DEPLOYMENT_NAME"], name="WeatherAgent"
        )
        query = "What's the weather like in Tokyo?"
        try:
            async with ChatAgent(
                # passing in the client is optional here, so if you take the agent_id from the portal
                # you can use it directly without the two lines above.
                chat_client=AzureAIAgentClient(project_client=client, agent_id=created_agent.id),
                instructions="You are a helpful weather agent.",
                tools=get_weather,
            ) as agent:
                result = await agent.run(query)
                print(f"Result: {result}\n")

                # query = "How is the weather in Seattle ?"
                tool_call = {
                    "type": "tool_call",
                    "tool_call_id": "call_CUdbkBfvVBla2YP3p24uhElJ",
                    "name": "fetch_weather",
                    "arguments": {"location": "Seattle"},
                }

                tool_definition = {
                    "id": "fetch_weather",
                    "name": "fetch_weather",
                    "description": "Fetches the weather information for the specified location.",
                    "parameters": {
                        "type": "object",
                        "properties": {"location": {"type": "string", "description": "The location to fetch weather for."}},
                    },
                }
                response = tool_call_accuracy(query=query, tool_calls=tool_call, tool_definitions=tool_definition)
                pprint(response)
                response_completeness_evaluator = ResponseCompletenessEvaluator(model_config=model_config)
                result = response_completeness_evaluator(
                    response=result,
                    ground_truth=result,
                )
                pprint(result)
                intent_resolution_evaluator = IntentResolutionEvaluator(model_config)
                # Success example. Intent is identified and understood and the response correctly resolves user intent
                result = intent_resolution_evaluator(
                    query=query,
                    response=result,
                )
                pprint(result)
                task_adherence_evaluator = TaskAdherenceEvaluator(model_config)
                result = task_adherence_evaluator(
                    query=query,
                    response=result,
                )
                pprint(result)
        finally:
            # Clean up the agent manually
            
            await client.agents.delete_agent(created_agent.id)


if __name__ == "__main__":
    asyncio.run(main())
```

- Save the file as `agtfrmkeval.py` and run the script.

```
python agtfrmkeval.py
```

- The output will show the result of the agent's response and the evaluation metrics from Azure AI Foundry services.
- here is the output example:

```
=== Azure AI Chat Client with Existing Agent ===
Result: The weather in Tokyo is currently sunny with a high of 22°C.

{'details': {'correct_tool_calls_made_by_agent': 0,
             'excess_tool_calls': {'details': [], 'total': 0},
             'missing_tool_calls': {'details': [{'missing_count': 1,
                                                 'tool_name': 'fetch_weather'}],
                                    'total': 1},
             'per_tool_call_details': [{'correct_calls_made_by_agent': 0,
                                        'correct_tool_percentage': 0.0,
                                        'tool_call_errors': 0,
                                        'tool_name': 'fetch_weather',
                                        'tool_success_result': 'fail',
                                        'total_calls_required': 1}],
             'tool_calls_made_by_agent': 1},
 'tool_call_accuracy': 2.0,
 'tool_call_accuracy_reason': "Let's think step by step: The user's last query "
                              "is 'What's the weather like in Tokyo?'. The "
                              "available tool is 'fetch_weather', which "
                              'fetches weather information for a specified '
                              'location. The agent made a single tool call to '
                              "'fetch_weather' with the argument 'location: "
                              "Seattle'. The correct parameter should have "
                              "been 'Tokyo', as explicitly stated in the "
                              "user's query. The tool call made is therefore "
                              'not grounded in the conversation and is '
                              'incorrect. Only one tool call was required, and '
                              'the agent made one, but with the wrong '
                              'parameter. There are no excess tool calls, but '
                              'the only tool call made is incorrect due to the '
                              'fabricated parameter. According to the '
                              'definitions, this is a Level 2: Partially '
                              'Relevant - Wrong Execution, because the tool '
                              'called is somewhat related, but the parameter '
                              'is not grounded in the conversation.',
 'tool_call_accuracy_result': 'fail',
 'tool_call_accuracy_threshold': 3}
[2025-11-09 08:36:20 - c:\Code\agentframework\msagentframework\.venv\Lib\site-packages\azure\ai\evaluation\_common\_experimental.py:79 - WARNING] Class ResponseCompletenessEvaluator: This is an experimental class, and may change at any time. Please see https://aka.ms/azuremlexperimental for more information.
{'response_completeness': 5,
 'response_completeness_reason': 'The response perfectly matches the ground '
                                 'truth, containing all necessary and relevant '
                                 'information without any omissions.',
 'response_completeness_result': 'pass',
 'response_completeness_threshold': 3}
[2025-11-09 08:36:22 - c:\Code\agentframework\msagentframework\.venv\Lib\site-packages\azure\ai\evaluation\_common\_experimental.py:79 - WARNING] Class IntentResolutionEvaluator: This is an experimental class, and may change at any time. Please see https://aka.ms/azuremlexperimental for more information.
[2025-11-09 08:36:22 - c:\Code\agentframework\msagentframework\.venv\Lib\site-packages\azure\ai\evaluation\_common\utils.py:575 - WARNING] Conversation history could not be parsed, falling back to original query: What's the weather like in Tokyo?
[2025-11-09 08:36:22 - c:\Code\agentframework\msagentframework\.venv\Lib\site-packages\azure\ai\evaluation\_common\utils.py:629 - WARNING] Empty agent response extracted, likely due to input schema change. Falling back to using the original response: {'response_completeness': 5, 'response_completeness_result': 'pass', 'response_completeness_threshold': 3, 'response_completeness_reason': 'The response perfectly matches the ground truth, containing all necessary and relevant information without any omissions.'}
{'intent_resolution': 1.0,
 'intent_resolution_reason': 'The user asked for the weather in Tokyo, but the '
                             "agent's response only contains a self-assessment "
                             'of response completeness and does not provide '
                             "any actual weather information. The user's "
                             'intent is not addressed at all.',
 'intent_resolution_result': 'fail',
 'intent_resolution_threshold': 3}
[2025-11-09 08:36:24 - c:\Code\agentframework\msagentframework\.venv\Lib\site-packages\azure\ai\evaluation\_common\_experimental.py:79 - WARNING] Class TaskAdherenceEvaluator: This is an experimental class, and may change at any time. Please see https://aka.ms/azuremlexperimental for more information.
[2025-11-09 08:36:24 - c:\Code\agentframework\msagentframework\.venv\Lib\site-packages\azure\ai\evaluation\_common\utils.py:575 - WARNING] Conversation history could not be parsed, falling back to original query: What's the weather like in Tokyo?
[2025-11-09 08:36:24 - c:\Code\agentframework\msagentframework\.venv\Lib\site-packages\azure\ai\evaluation\_common\utils.py:638 - WARNING] Agent response could not be parsed, falling back to original response: {'intent_resolution': 1.0, 'intent_resolution_result': 'fail', 'intent_resolution_threshold': 3, 'intent_resolution_reason': "The user asked for the weather in Tokyo, but the agent's response only contains a self-assessment of response completeness and does not provide any actual weather information. The user's intent is not addressed at all."}
{'task_adherence': 1.0,
 'task_adherence_reason': 'The assistant failed to provide any weather '
                          'information for Tokyo and instead outputted a '
                          'self-assessment. No tools were used or required, '
                          "but the user's request was completely unaddressed.",
 'task_adherence_result': 'fail',
 'task_adherence_threshold': 3}
 ```

- Evaluations can be local or can be run using Azure AI Foundry evaluation services.
- This example demonstrates how to evaluate locally using Azure AI Foundry evaluation classes.

## Conclusion

- This evaluation demonstrates the integration of Microsoft Agent Framework with Azure AI Foundry services.
- The agent was able to perform its task effectively, and the evaluation metrics provided insights into its performance.
- Further improvements can be made based on the evaluation results to enhance the agent's capabilities and accuracy.