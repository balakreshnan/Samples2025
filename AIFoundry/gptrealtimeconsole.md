# Azure AI Foundry - GPT-Realtime - python code

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
from azure.identity.aio import DefaultAzureCredential, get_bearer_token_provider
import numpy as np
import sounddevice as sd
from dotenv import load_dotenv
load_dotenv()
```

- set the variables
- function to display output

```
def handle_response_done(event):
    """
    Parse a ResponseDoneEvent and display transcript + usage details.
    """
    response = event.response

    print("=== ✅ Response Done ===")
    print(f"Response ID: {response.id}")
    print(f"Conversation ID: {response.conversation_id}")
    print(f"Status: {response.status}")
    print(f"Modalities: {response.modalities}")
    print(f"Voice: {response.voice}")

    # Extract transcript(s) from output items
    for item in response.output:
        if hasattr(item, "content"):
            for c in item.content:
                if c.type == "audio" and c.transcript:
                    print("\n🗣️ Final Transcript:")
                    print(c.transcript)

    # Token usage details
    usage = response.usage
    print("\n=== 📊 Token Usage ===")
    print(f" Input tokens: {usage.input_tokens}")
    print(f" Output tokens: {usage.output_tokens}")
    print(f" Total tokens: {usage.total_tokens}")
    print(f" Audio output tokens: {usage.output_token_details.audio_tokens}")
    print(f" Text output tokens: {usage.output_token_details.text_tokens}")
```

- here is the function to call the gpt-realtime model

```
async def main() -> None:
    """
    When prompted for user input, type a message and hit enter to send it to the model.
    Enter "q" to quit the conversation.
    """
    audio_chunks = []

    client = AsyncAzureOpenAI(
        azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
        api_key=os.getenv("AZURE_OPENAI_KEY"),
        api_version="2024-10-01-preview",
    )
    async with client.beta.realtime.connect(
        model="gpt-realtime",  # deployment name of your model
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
                # qprint('Event: ', event)
                if event.type == "response.audio.delta":
                    # audio_chunks.append(event.delta)
                    audio_chunks.append(base64.b64decode(event.delta))
                    # print(event.delta, flush=True, end="")
                    audio_data = base64.b64decode(event.delta)
                    print(f"Received {len(audio_data)} bytes of audio data.")
                    
                elif event.type == "response.audio.delta":
                    audio_data = base64.b64decode(event.delta)
                    print(f"Received {len(audio_data)} bytes of audio data.")
                elif event.type == "response.audio.delta":

                    audio_data = base64.b64decode(event.delta)
                    print(f"Received {len(audio_data)} bytes of audio data.")
                elif event.type == "response.audio_transcript.done":
                    # print(f"Received text delta: {event.delta}")
                    print()
                elif event.type == "response.audio_transcript.done":
                    for c in event.item.content:
                        if c.type == "audio" and c.transcript:
                            print(f"[Final Transcript] {c.transcript}")
                    print()
                elif event.type == "response.done":
                    handle_response_done(event)
                    print("✅ Response complete, playing audio...")

                    # Combine all PCM16 chunks
                    audio_bytes = b"".join(audio_chunks)

                    # Convert to numpy array for sounddevice
                    audio_np = np.frombuffer(audio_bytes, dtype=np.int16)

                    # Play at 24 kHz, mono
                    sd.play(audio_np, samplerate=24000, blocking=True)
                    sd.wait()

                    print("🔊 Done playing audio")
                    audio_chunks.clear()  # Clear chunks for next response
                    break
```

- run the main function

```
if __name__ == "__main__":
    asyncio.run(main())
```

- run the code
- output

```
 Final Transcript:
Sure! Quantum computing uses quantum bits (qubits) that can be in multiple states at once, thanks to superposition and entanglement. This allows quantum computers to solve certain problems, like factoring large numbers and simulating molecules, much faster than classical computers.

=== 📊 Token Usage ===
 Input tokens: 341
 Output tokens: 433
 Total tokens: 774
 Audio output tokens: 356
 Text output tokens: 77
✅ Response complete, playing audio...
🔊 Done playing audio
```

- Done

## Conclusion

- we have successfully run the gpt-realtime model using python sdk
- there was processing syntax error with microsoft documentation