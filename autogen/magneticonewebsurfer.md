# Microsoft Autogen 0.4 - MultiModel Web Surfer - MagenticOne

## Introduciton

- Idea here is to show how to use autogen 0.4 and summarize web sites or get latest information from multiple websites using MagenticOne
- This is a simple example to show how to use autogen
- This is not a production ready code
- Ability to serialize and deserialize objects
- We are using MagenticOneGroupChat for team.

## Pre-requisites

- Install autogen 0.4 and above
- Azure Subscription
- Azure Open AI
- Python 3.12 and above
- install MagenticOne
- install pip install autogen-ext[web-surfer]

## Steps

- Loop all the PDF files in a folder and summarize each PDF file
- installation needed

```
pip install autogen-agentchat autogen-ext[magentic-one,openai]

playwright install --with-deps chromium

autogen-ext[web-surfer]
```

- Agent will take a input and perform operation using web browser
- Now import all the libraries

```
import asyncio
import sys
from autogen_ext.models.openai import AzureOpenAIChatCompletionClient
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import MagenticOneGroupChat, RoundRobinGroupChat
from autogen_agentchat.ui import Console
from autogen_ext.agents.web_surfer import MultimodalWebSurfer
import os

from dotenv import load_dotenv
```

- Load environment variables

```
load_dotenv()
```

- Configure Model Client

```
if sys.platform == "win32":
    asyncio.set_event_loop_policy(asyncio.WindowsProactorEventLoopPolicy())

model_client = AzureOpenAIChatCompletionClient(model="gpt-4o",
                                               azure_deployment="gpt-4o-2", 
                                               azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"), 
                                               api_key=os.getenv("AZURE_OPENAI_API_KEY"), 
                                               api_version="2024-10-21",
                                               temperature=0.0,
                                               seed=42,
                                               maz_tokens=4096)
```

- now to create MagenticOneGroupChat
- Using Multi modal web surfer

```
async def main() -> None:

    surfer = MultimodalWebSurfer(
        "MultimodalWebSurfer",
        model_client=model_client,
        downloads_folder="./downs",
        debug_dir="./debug",
        headless = False,
    )
    team = MagenticOneGroupChat([surfer], model_client=model_client, max_turns=3)
    # Define a team
    # team = RoundRobinGroupChat([surfer], max_turns=3)
    #await Console(team.run_stream(task="Navigate to the AutoGen readme on GitHub."))
    # await Console(team.run_stream(task="Summarize latest updates from Accenture newsroowm."))
    await Console(team.run_stream(task="Summarize latest news from venture beat all things in AI."))
    await surfer.close()


if __name__ == "__main__":
    if sys.platform == "win32":
        asyncio.set_event_loop_policy(asyncio.WindowsProactorEventLoopPolicy())
    asyncio.run(main())
```

- Run the code and see the output
- i have few sample question in the code
- here is the output

```
---------- MagenticOneOrchestrator ----------
Here's a summary of the latest AI news from VentureBeat:

1. **MongoDB's AI Solutions**: MongoDB is tackling AI hallucination issues by using advanced rerankers and embedding models to improve data accuracy.  

2. **xAI's Grok 3 Model**: xAI's new Grok 3 model has faced criticism for blocking sources that identify Elon Musk and Donald Trump as top misinformation spreaders.

3. **Browser-Use Agents**: Convergence's Proxy is gaining attention for outperforming OpenAI's Operator in the realm of browser-use agents.

4. **Escape.ai Platform**: John Gaeta has introduced Escape.ai, a platform designed for emerging entertainment technologies.

5. **Hume's Octave Model**: Hume has launched Octave, a text-to-speech model capable of generating custom AI voices with adjustable emotions.

6. **OpenAI's Deep Research Access**: OpenAI is expanding Deep Research access to Plus users, intensifying competition with DeepSeek and Claude.       

7. **Google's Gemini Code Assist**: Google has released Gemini Code Assist, offering 180,000 free code completions per month to developers.

8. **Other Developments**: The page also covers data security strategies, Adobe's new AI-powered Photoshop mobile app, and Quantum Machines' $170 million funding to advance quantum computing.

These stories highlight a wide range of AI advancements and challenges, from security and infrastructure to entertainment and development tools. 
```

- Try different questions.