# Azure AI Foundry Responsible AI Agent SDK approach

## Introduction

- Create a basic RAG and RAI agent using the Azure AI Foundry Agent SDK approach
- Building a manufacturing compliance agent using the Azure AI Foundry Agent SDK approach
- Using OSHA and NIST to obtain best practices for manufacturing compliance for PPE, Cybersecurity, and other manufacturing compliance requirements.
- Code first approach
- I am reusing an existing index i have created using Azure AI Foundry UI - talk to your data
- For the MFG compliance data create batch Responsible AI agent to evaluate
- This evaluation can be part of Gen AI ops pipeline
- Evaluation can also be part of Development practices.
- This is a important step to make sure the input and output are safe and secure.
- Using SDK approach to develop and implement and operations is through Azure AI Foundry Studio UI
- Also enables Tracing for agents.

## Prerequisites

- Azure Subscription
- Azure AI Foundry Resource
- Azure AI Foundry project
- Azure Open AI in east us2 and sweden for RAI
- Azure AI Search
- Docs and Index are already created in Azure AI Foundry
- Using Visual Studio Code in local Laptop with python environment 3.13 and above
- Dataset i used is custom data with groundtruth and response data

## Steps

- install Azure AI Foundry SDK library

```
pip install -U azure-ai-projects
pip install -U azure-ai-evaluation
pip install -U azure-ai-inference
pip install -U azure-monitor-opentelemetry
pip install -U opentelemetry-instrumentation-openai-v2
pip install -U azure-core-tracing-opentelemetry
```

- First import all the libraries
- Load all the environment variables
- Make sure data is loaded using Azure AI Foundry UI into Azure AI Search
- i curated a set of documents for manufacturing compliance from our OSHA and NIST data governement web sites.
- Make sure Environment variables are set for Azure AI Foundry and Azure Open AI, Azure AI Search
- i have created 2 agents one for using inbuilt AI Search tool agent.
- Other one is custom function agent for Responsible AI metrics.
- i have also included all the metrics available at the time of writing this code.
- All metrics of Responsible AI are logged in Azure AI foundry for Review and Audit, Compliance and Governance.

### Code

```python
import json
import os
import time
from typing import Any, Callable, Set, Dict, List, Optional
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential
from azure.ai.projects.models import AzureAISearchTool
# [START enable_tracing]
from opentelemetry import trace
from azure.monitor.opentelemetry import configure_azure_monitor
from pprint import pprint
from azure.ai.evaluation import evaluate, AzureAIProject, AzureOpenAIModelConfiguration, F1ScoreEvaluator
from azure.ai.evaluation import ProtectedMaterialEvaluator, IndirectAttackEvaluator, RetrievalEvaluator
from azure.ai.evaluation.simulator import AdversarialSimulator, AdversarialScenario, IndirectAttackSimulator
from azure.identity import DefaultAzureCredential
from azure.ai.evaluation import RelevanceEvaluator
from azure.ai.evaluation import (
    ContentSafetyEvaluator,
    RelevanceEvaluator,
    CoherenceEvaluator,
    GroundednessEvaluator,
    FluencyEvaluator,
    SimilarityEvaluator,
    ViolenceEvaluator,
    SexualEvaluator,
    SelfHarmEvaluator,
    HateUnfairnessEvaluator,
)
from mfgdata import extractmfgresults, extracttop5questions
from azure.ai.evaluation import BleuScoreEvaluator, GleuScoreEvaluator, RougeScoreEvaluator, MeteorScoreEvaluator, RougeType
from azure.ai.projects.models import FunctionTool, RequiredFunctionToolCall, SubmitToolOutputsAction, ToolOutput, ToolSet
from dotenv import load_dotenv

# Load .env file
load_dotenv()

connection_string = os.environ["PROJECT_CONNECTION_STRING_EASTUS2"] 

# print(f"Connection string: {connection_string}")

project_client = AIProjectClient.from_connection_string(
    credential=DefaultAzureCredential(),
    conn_str=connection_string,
)

# Enable Azure Monitor tracing
application_insights_connection_string = project_client.telemetry.get_connection_string()
if not application_insights_connection_string:
    print("Application Insights was not enabled for this project.")
    print("Enable it via the 'Tracing' tab in your AI Foundry project page.")
    exit()
configure_azure_monitor(connection_string=application_insights_connection_string)

scenario = os.path.basename(__file__)
tracer = trace.get_tracer(__name__)

# Function to parse the JSON data
def parse_json(data):
    # Print Relevance Score
    print(f"Overall GPT Relevance: {data.get('relevance.gpt_relevance', 'N/A')}")
    
    # Print Rows
    rows = data.get('rows', [])
    print("\nRows:")
    for row in rows:
        context = row.get('inputs.context')
        query = row.get('inputs.query')
        response = row.get('inputs.response')
        output = row.get('outputs.output')
        relevance = row.get('outputs.relevance.gpt_relevance')
        
        print(f"Context: {context}")
        print(f"Query: {query}")
        print(f"Response: {response}")
        print(f"Output: {output}")
        print(f"Relevance: {relevance}")
        print("-" * 50)

class AnswerLengthEvaluator:
    def __init__(self):
        pass
    # A class is made a callable my implementing the special method __call__
    def __call__(self, *, answer: str, **kwargs):
        return {"answer_length": len(answer)}
    
def evalmetrics(query: str) -> str:
    
    returntxt = ""
    # Load .env file
    # load_dotenv()
    #citationtxt = extractrfpresults("Provide summary of Resources for Railway projects with 200 words?")

    #print('Citation Text:', citationtxt)
    azure_endpoint = os.getenv("AZURE_OPENAI_ENDPOINT")
    api_key = os.getenv("AZURE_OPENAI_API_KEY")
    azure_deployment = os.getenv("AZURE_OPENAI_DEPLOYMENT")
    api_version = os.getenv("AZURE_OPENAI_API_VERSION")

    model_config = {
        "azure_endpoint": azure_endpoint,
        "api_key": api_key,
        "azure_deployment": azure_deployment,
        "api_version": api_version,
    }


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
    
    azure_ai_project_dict = {
        "subscription_id": subscription_id,
        "resource_group_name": resource_group_name,
        "project_name": project_name,
        "azure_credential": credential
    }
    
    # prompty_path = os.path.join("./", "rfp.prompty")
    content_safety_evaluator = ContentSafetyEvaluator(azure_ai_project=azure_ai_project_dict, credential=credential)
    relevance_evaluator = RelevanceEvaluator(model_config)
    coherence_evaluator = CoherenceEvaluator(model_config)
    groundedness_evaluator = GroundednessEvaluator(model_config)
    fluency_evaluator = FluencyEvaluator(model_config)
    # similarity_evaluator = SimilarityEvaluator(model_config)
    f1_evaluator = F1ScoreEvaluator()
    bleu_evaluator = BleuScoreEvaluator()
    gleu_evaluator = GleuScoreEvaluator()
    meteor_evaluator = MeteorScoreEvaluator(alpha=0.8)
    rouge_evaluator = RougeScoreEvaluator(rouge_type=RougeType.ROUGE_4)
    indirect_attack_eval = IndirectAttackEvaluator(azure_ai_project=azure_ai_project_dict, credential=credential)
    protected_material_eval = ProtectedMaterialEvaluator(azure_ai_project=azure_ai_project_dict, credential=credential)
    hate_unfairness_eval = HateUnfairnessEvaluator(azure_ai_project=azure_ai_project_dict, credential=credential)
    answer_length_evaluator = AnswerLengthEvaluator()

    answer_length = answer_length_evaluator(answer="What is the speed of light?")

    print(answer_length)

    results = evaluate(
        evaluation_name="mfgcomplevaluation",
        data="datarfp.jsonl",
        target=extractmfgresults,
        #evaluators={
        #    "relevance": relevance_evaluator,
        #},
        #evaluator_config={
        #    "relevance": {"response": "${target.response}", "context": "${data.context}", "query": "${data.query}"},
        #},
        evaluators={
            "content_safety": content_safety_evaluator,
            "coherence": coherence_evaluator,
            "relevance": relevance_evaluator,
            "groundedness": groundedness_evaluator,
            "fluency": fluency_evaluator,
        #    "similarity": similarity_evaluator,
            "f1": f1_evaluator,
            "bleu": bleu_evaluator,
            "gleu": gleu_evaluator,
            "meteor": meteor_evaluator,
            "rouge": rouge_evaluator,
            "indirect_attack": indirect_attack_eval,
            "protected_material": protected_material_eval,
            "hate_unfairness": hate_unfairness_eval,
            # "answer_length": answer_length_evaluator,
        },        
        evaluator_config={
            "content_safety": {"query": "${data.query}", "response": "${target.response}"},
            "coherence": {"response": "${target.response}", "query": "${data.query}"},
            "relevance": {"response": "${target.response}", "context": "${data.context}", "query": "${data.query}"},
            "groundedness": {
                "response": "${target.response}",
                "context": "${data.context}",
                "query": "${data.query}",
            },
            "fluency": {"response": "${target.response}", "context": "${data.context}", "query": "${data.query}"},
            "similarity": {"response": "${target.response}", "context": "${data.context}", "query": "${data.query}"},
            "f1": {"response": "${target.response}", "ground_truth": "${data.ground_truth}"},
            "bleu": {"response": "${target.response}", "ground_truth": "${data.ground_truth}"},
            "gleu": {"response": "${target.response}", "ground_truth": "${data.ground_truth}"},
            "meteor": {"response": "${target.response}", "ground_truth": "${data.ground_truth}"},
            "rouge": {"response": "${target.response}", "ground_truth": "${data.ground_truth}"},
            "indirect_attack": {"query": "${data.query}", "response": "${target.response}"},
            "protected_material": {"query": "${data.query}", "response": "${target.response}"},
            "hate_unfairness": {"query": "${data.query}", "response": "${target.response}"},
            # "answer_length": {"answer": "${target.response}"},
        },
        azure_ai_project=azure_ai_project,
        output_path="./rsoutputmetrics.json",
    )
    # pprint(results)
    # parse_json(results)
    print("Done")
    returntxt = "Completed Evaluation"
    return returntxt

def send_email_using_recipient_name(recipient: str, subject: str, body: str) -> str:
    """
    Sends an email with the specified subject and body to the recipient.

    :param recipient (str): Name of the recipient.
    :param subject (str): Subject of the email.
    :param body (str): Body content of the email.
    :return: Confirmation message.
    :rtype: str
    """
    # In a real-world scenario, you'd use an SMTP server or an email service API.
    # Here, we'll mock the email sending.
    print(f"Sending email to {recipient}...")
    print(f"Subject: {subject}")
    print(f"Body:\n{body}")

    message_json = json.dumps({"message": f"Email successfully sent to {recipient}."})
    return message_json

def send_email(recipient: str, subject: str, body: str) -> str:
    """
    Sends an email with the specified subject and body to the recipient.

    :param recipient (str): Email address of the recipient.
    :param subject (str): Subject of the email.
    :param body (str): Body content of the email.
    :return: Confirmation message.
    :rtype: str
    """
    # In a real-world scenario, you'd use an SMTP server or an email service API.
    # Here, we'll mock the email sending.
    print(f"Sending email to {recipient}...")
    print(f"Subject: {subject}")
    print(f"Body:\n{body}")

    message_json = json.dumps({"message": f"Email successfully sent to {recipient}."})
    return message_json


def main():
    with tracer.start_as_current_span(scenario):
        conn_list = project_client.connections.list()
        conn_id = ""
        for conn in conn_list:
            if conn.connection_type == "CognitiveSearch":
                print(f"Connection ID: {conn.id}")
                conn_id = conn.id

        ai_search = AzureAISearchTool(conn_id, "mfggptdata")
        #ai_search.add_index(conn_id, "mfggptdata")

        agent = project_client.agents.create_agent(
            model="gpt-4o",
            name="MFG-Compliance-Agent",
            instructions="You are a helpful assistant",
            tools=ai_search.definitions,
            tool_resources = ai_search.resources,
        )
        print(f"Created agent, ID: {agent.id}")

        # Create a thread
        thread = project_client.agents.create_thread()
        print(f"Created thread, thread ID: {thread.id}")
        
        # Create a message
        message = project_client.agents.create_message(
            thread_id=thread.id,
            role="user",
            content="what are the personal protection i should consider in manufacturing?",
        )
        print(f"Created message, message ID: {message.id}")
            
        # Run the agent
        run = project_client.agents.create_and_process_run(thread_id=thread.id, assistant_id=agent.id)
        print(f"Run Manufacturing agent finished with status: {run.status}")
        
        if run.status == "failed":
            # Check if you got "Rate limit is exceeded.", then you want to get more quota
            print(f"Run failed: {run.last_error}")

        # Get messages from the thread 
        messages = project_client.agents.list_messages(thread_id=thread.id)
        # print(f"Messages: {messages}")
            
        assistant_message = ""
        for message in messages.data:
            if message["role"] == "assistant":
                assistant_message = message["content"][0]["text"]["value"]

        # Get the last message from the sender
        print(f"Assistant response: {assistant_message}")

        #now lets evaluate the output
        # evalmetrics("evaluation")

        # Now evaluate using agent for metrics

        # https://github.com/Azure/azure-sdk-for-python/blob/azure-ai-projects_1.0.0b5/sdk/ai/azure-ai-projects/samples/agents/user_functions.py

        print(f"Here is the Start of Responsible AI Agent")
        user_functions: Set[Callable[..., Any]] = {
            evalmetrics,
        }
        functions = FunctionTool(user_functions)
        toolset = ToolSet()
        toolset.add(functions)

        agent = project_client.agents.create_agent(
            model="gpt-4o",
            name="ResponsibileAI-Agent",
            instructions="You are a Responsible AI assistant. Run the toolset to evaluate the output.",
            toolset=toolset,
        )
        print(f"Created agent, ID: {agent.id}")

        thread = project_client.agents.create_thread()
        print(f"Created thread, ID: {thread.id}")

        message = project_client.agents.create_message(
            thread_id=thread.id,
            role="user",
            content="what are the personal protection i should consider in manufacturing?",
        )
        print(f"Created message, ID: {message.id}")

        run = project_client.agents.create_run(thread_id=thread.id, assistant_id=agent.id)
        print(f"Created run, ID: {run.id}")

        while run.status in ["queued", "in_progress", "requires_action"]:
            time.sleep(1)
            run = project_client.agents.get_run(thread_id=thread.id, run_id=run.id)

            if run.status == "requires_action" and isinstance(run.required_action, SubmitToolOutputsAction):
                tool_calls = run.required_action.submit_tool_outputs.tool_calls
                if not tool_calls:
                    print("No tool calls provided - cancelling run")
                    project_client.agents.cancel_run(thread_id=thread.id, run_id=run.id)
                    break

                tool_outputs = []
                for tool_call in tool_calls:
                    if isinstance(tool_call, RequiredFunctionToolCall):
                        try:
                            print(f"Executing tool call: {tool_call}")
                            output = functions.execute(tool_call)
                            tool_outputs.append(
                                ToolOutput(
                                    tool_call_id=tool_call.id,
                                    output=output,
                                )
                            )
                        except Exception as e:
                            print(f"Error executing tool_call {tool_call.id}: {e}")

                print(f"Tool outputs: {tool_outputs}")
                if tool_outputs:
                    project_client.agents.submit_tool_outputs_to_run(
                        thread_id=thread.id, run_id=run.id, tool_outputs=tool_outputs
                    )

            print(f"Current RAI Agent run status: {run.status}")
        print(f"Run RAI Agent completed with status: {run.status}")

        # now runing the evaluation manually
        # evalmetrics("evaluation")


if __name__ == "__main__":
    main()
```

- Run the code

```
python aifoundryraiagent.py
```

- Wait for the code to complete
- Last few lines of output

```
Evaluation results saved to "C:\Code\agifoundation1\rsoutputmetrics.json".

Done
Tool outputs: [{'tool_call_id': 'call_lSEWHrhELXbxHy6u7L5qoKV9', 'output': 'Completed Evaluation'}]
Current RAI Agent run status: RunStatus.REQUIRES_ACTION
Current RAI Agent run status: RunStatus.IN_PROGRESS
Current RAI Agent run status: RunStatus.IN_PROGRESS
Current RAI Agent run status: RunStatus.COMPLETED
Run RAI Agent completed with status: RunStatus.COMPLETED
```

- Now go to Azure AI Foundry UI and check the results

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundryraiagent-1.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundryraiagent-2.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundryraiagent-3.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundryraiagent-4.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundryraiagent-5.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundryraiagent-6.jpg 'RagChat')

- Tracing output
- Provides application lineage

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundryraiagent-7.jpg 'RagChat')
![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/aifoundryraiagent-8.jpg 'RagChat')

- Dataset i used is custom data with groundtruth and response data
- Responses were from GPT 4o model.
- Hope this helps.
- Happy coding!!!