# Azure AI Foundry - GPT-Realtime streaming - python code

## Introduction

- Azure Subscription
- Azure AI Foundry
- Azure Storage
- Deploy gpt-realtime model
- python > 3.13
- https://learn.microsoft.com/en-us/azure/ai-foundry/openai/realtime-audio-quickstart?tabs=api-key%2Cwindows&pivots=programming-language-python
- using python sdk

## Code

- install the python

```
azure-identity==1.22.0
certifi==2025.4.26
cffi==1.17.1
cryptography==44.0.3
numpy==2.2.5
pycparser==2.22
requests==2.32.3
sounddevice==0.5.1
typing_extensions==4.13.2
urllib3==2.4.0
websocket-client==1.8.0
openai[realtime]
```

- import the libraries

```
import os
import base64
import asyncio
from openai import AsyncAzureOpenAI
import numpy as np
import sounddevice as sd
from dotenv import load_dotenv

load_dotenv()
```

- here is the code

```
async def main() -> None:
    client = AsyncAzureOpenAI(
        azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
        api_key=os.getenv("AZURE_OPENAI_KEY"),
        api_version="2024-10-01-preview",
    )

    # Open a sounddevice output stream (24kHz mono)
    stream = sd.OutputStream(samplerate=24000, channels=1, dtype="int16")
    stream.start()

    async with client.beta.realtime.connect(
        model="gpt-realtime",
    ) as connection:
        await connection.session.update(session={"output_modalities": ["text", "audio"]})

        while True:
            user_input = input("Enter a message: ")
            if user_input == "q":
                break

            await connection.conversation.item.create(
                item={
                    "type": "message",
                    "role": "user",
                    "content": [{"type": "input_text", "text": user_input}],
                }
            )
            await connection.response.create()

            async for event in connection:
                if event.type == "response.audio.delta":
                    audio_data = base64.b64decode(event.delta)
                    audio_np = np.frombuffer(audio_data, dtype=np.int16)
                    stream.write(audio_np)  # 🔊 play immediately

                elif event.type == "response.audio_transcript.delta":
                    print(event.delta, flush=True, end="")

                elif event.type == "response.audio_transcript.done":
                    # for c in event.item.content:
                    #     if c.type == "audio" and c.transcript:
                    #         print(f"\n[Final Transcript] {c.transcript}")
                    print()

                elif event.type == "response.done":
                    print("✅ Response complete\n")
                    break

    stream.stop()
    stream.close()
```

- run the code

```
if __name__ == "__main__":
    asyncio.run(main())
```

- save the code
- run the code

```
python gptrealtimestrream.py
```

- output

```
Enter a message: how are you?
Hey, I'm doing great! Thanks for asking. Just here and ready to help you out with whatever you need. How about you? How’s it going on your end?
=== ✅ Response Done ===
Response ID: resp_CG5NjDbHzJH69OQKOprbx
Conversation ID: conv_CG5NgzGPQ7K98MmuI5EFk
Status: completed
Modalities: ['audio', 'text']
Voice: alloy

🗣️ Final Transcript:
Hey, I'm doing great! Thanks for asking. Just here and ready to help you out with whatever you need. How about you? How’s it going on your end?

=== 📊 Token Usage ===
 Input tokens: 119
 Output tokens: 206
 Total tokens: 325
 Audio output tokens: 155
 Text output tokens: 51
✅ Response complete
```

## Conclusion

- we have successfully run the gpt-realtime streaming model using python sdk
- there was processing syntax error with microsoft documentation