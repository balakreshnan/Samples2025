# Build a Voice enabled Azure AI Foundry Agent using GPT realtime Models

## Introduction

- Build a voice enabled agent using Azure AI Foundry and GPT realtime models
- Ability to converse with the agent using voice
- Converse like human - back and forth
- Also uploaded documents and created a index to use as knowledge base in the agent
- Agent will respond based on the knowledge base
- Only for learning purpose
- Models, syntax might change based on the latest updates SDK's.

## Prerequisites

- Azure Subscription
- Azure AI Foundry
- Azure Storage
- Deploy gpt-realtime model
- python > 3.13
- Azure AI search
- Create index with sample documents
- I used my custom documents to create the index
- In your case please change it to use yours.
- I won't cover how to create index and agent in this document
- We will focus on using existing agent but providing a voice interface

## Demo

- Here is the demo of the application

[![Watch the video](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/voiceagentex-1.jpg)](https://youtube.com/shorts/zyAJrv-xzFI?feature=share)

## Code

- This application is conbination of flask and html javascript
- We are using web socket to connect to the backend voice Azure AI Foundry agent
- Install the python packages

```
azure-ai-projects==1.1.0b3
azure-ai-agents==1.2.0b3
streamlit
openai
uv
python-dotenv
flask
flask-socketio
aiohttp==3.11.18
azure-core==1.34.0
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
pyaudio
```

- Create a environment file .env

```
AZURE_SUBSCRIPTION_ID="xxxxxxxx"
AZURE_VOICE_LIVE_ENDPOINT="https://aifoudnryresource.services.ai.azure.com"
AI_FOUNDRY_PROJECT_NAME="projectname"
AI_FOUNDRY_AGENT_ID="asst_xxxxxxxx"
AZURE_VOICE_LIVE_API_VERSION="2025-05-01-preview"
AZURE_VOICE_LIVE_API_KEY="xxxxx" # Only required if using API key authentication
AZURE_VOICE_LIVE_DEPLOYMENT="gpt-4o-realtime-preview"
```

- Make sure you fill based on your environment
- First lets create the html File for the UI
- Code for html - [templates/index.html](https://github.com/balakreshnan/intsalesadvisor/blob/main/templates/voice_live_web.html)
- Now lels create the back end code to process connect to agent and process the voice
- Here is the link - https://github.com/balakreshnan/intsalesadvisor/blob/main/voice_live_web.py
- above the dependencies are required for the actual python web app to execute
- Now lets write the main server code, this code will enable the web socket connection and process the voice
- Will also allow us to load the html page
- Here is the code

```
import os
import uuid
import json
import time
import base64
import logging
import threading
import numpy as np
import queue
import asyncio
try:
    import websocket as ws_client
except ImportError:
    print("websocket-client package not installed. Please run: pip install websocket-client")
    ws_client = None
from datetime import datetime
from collections import deque

from flask import Flask, render_template, request
from flask_socketio import SocketIO, emit
from dotenv import load_dotenv
from azure.identity import DefaultAzureCredential

# Import the existing classes from voice_live_web.py
import sys
sys.path.append(os.path.dirname(os.path.abspath(__file__)))
from voice_live_web import AzureVoiceLive, VoiceLiveConnection, AudioPlayerAsync

load_dotenv()

# Initialize Flask app
try:
    app = Flask(__name__)
    app.config['SECRET_KEY'] = 'your-secret-key'
    socketio = SocketIO(app, cors_allowed_origins="*")
    logger = logging.getLogger(__name__)
except Exception as e:
    print(f"Error initializing Flask app: {e}")
    print("Please ensure Flask and Flask-SocketIO are installed: pip install flask flask-socketio")
    exit(1)

# Global variables for managing connections
active_connections = {}

class WebVoiceLiveSession:
    """Manages a Voice Live session for web clients"""
    
    def __init__(self, session_id: str):
        """Initialize state for a single web client session."""
        self.session_id = session_id
        self.connection = None
        self.audio_player = AudioPlayerAsync()
        self.is_active = False
        self.response_in_progress = False

        # Audio delta aggregation for smoother client playback
        self._audio_delta_accum = []      # base64 delta fragments
        self._audio_accum_bytes = 0       # accumulated decoded bytes
        self._audio_first_delta_time = None

        # Tunable aggregation parameters (can be adjusted for latency vs smoothness)
        self.AGG_TARGET_MS = int(os.getenv("VOICE_AGG_TARGET_MS", "140"))  # desired buffered ms before emit
        self.AGG_MAX_WAIT_MS = int(os.getenv("VOICE_AGG_MAX_WAIT_MS", "80"))  # max wait after first small chunk
        self.AGG_MAX_BYTES = (24000 // 1000) * 2 * 200  # up to ~200ms @24kHz 16-bit mono
        self._assumed_sample_rate = int(os.getenv("VOICE_ASSUMED_SAMPLE_RATE", "24000"))
        
    def start_session(self):
        """Initialize Voice Live API connection"""
        if ws_client is None:
            raise ImportError("websocket-client package not installed. Please run: pip install websocket-client")
            
        try:
            # Get credentials
            endpoint = os.environ.get("AZURE_VOICE_LIVE_ENDPOINT")
            agent_id = os.environ.get("AI_FOUNDRY_AGENT_ID")
            project_name = os.environ.get("AI_FOUNDRY_PROJECT_NAME")
            api_version = os.environ.get("AZURE_VOICE_LIVE_API_VERSION", "2025-05-01-preview")
            
            credential = DefaultAzureCredential()
            scopes = "https://ai.azure.com/.default"
            token = credential.get_token(scopes)
            
            # Create client and connection
            client = AzureVoiceLive(
                azure_endpoint=endpoint,
                api_version=api_version,
                token=token.token,
            )
            
            self.connection = client.connect(
                project_name=project_name,
                agent_id=agent_id,
                agent_access_token=token.token
            )
            
            # Configure session for real-time voice
            session_update = {
                "type": "session.update",
                "session": {
                    "turn_detection": {
                        "type": "azure_semantic_vad",
                        "threshold": 0.3,
                        "prefix_padding_ms": 200,
                        "silence_duration_ms": 200,
                        "remove_filler_words": False,
                        "end_of_utterance_detection": {
                            "model": "semantic_detection_v1",
                            "threshold": 0.01,
                            "timeout": 2,
                        },
                    },
                    "input_audio_noise_reduction": {
                        "type": "azure_deep_noise_suppression"
                    },
                    "input_audio_echo_cancellation": {
                        "type": "server_echo_cancellation"
                    },
                    "voice": {
                        "name": "en-US-Ava:DragonHDLatestNeural",
                        "type": "azure-standard",
                        "temperature": 0.8,
                    },
                },
                "event_id": ""
            }
            
            self.connection.send(json.dumps(session_update))
            self.is_active = True
            
            # Start listening for responses
            threading.Thread(target=self._listen_for_responses, daemon=True).start()
            
            socketio.emit('session_started', {'status': 'success'}, room=self.session_id)
            print(f"Session {self.session_id} started successfully")
            
        except Exception as e:
            print(f"Error starting session {self.session_id}: {e}")
            socketio.emit('session_error', {'error': str(e)}, room=self.session_id)
    
    def _listen_for_responses(self):
        """Listen for responses from Voice Live API"""
        while self.is_active and self.connection:
            try:
                raw_event = self.connection.recv()
                if raw_event is None:
                    time.sleep(0.01)
                    continue
                
                event = json.loads(raw_event)
                event_type = event.get("type")
                
                # Handle different event types
                if event_type == "session.created":
                    session = event.get("session")
                    print(f"Session created: {session.get('id')}")
                    
                elif event_type == "conversation.item.input_audio_transcription.completed":
                    transcript = event.get("transcript", "")
                    socketio.emit('transcript', {'text': transcript}, room=self.session_id)
                    
                elif event_type == "response.text.done":
                    agent_text = event.get("text", "")
                    socketio.emit('agent_text', {'text': agent_text}, room=self.session_id)
                    
                elif event_type == "response.audio_transcript.done":
                    agent_audio_text = event.get("transcript", "")
                    socketio.emit('agent_audio_transcript', {'text': agent_audio_text}, room=self.session_id)
                    
                elif event_type == "response.audio.delta":
                    # Send audio data back to client for playback
                    audio_data = event.get("delta", "")
                    if audio_data:
                        try:
                            self._accumulate_or_emit_delta(audio_data)
                        except Exception as agg_err:
                            print(f"Aggregation error, falling back direct emit: {agg_err}")
                            socketio.emit('audio_chunk', {'audio': audio_data}, room=self.session_id)
                    else:
                        print("Received empty audio delta")
                        
                elif event_type == "response.audio.done":
                    print("Audio response completed")
                    # Flush any remaining aggregated audio
                    self._flush_audio_accum(force=True)
                    socketio.emit('response_audio_done', {}, room=self.session_id)
                    
                elif event_type == "response.done":
                    print("Full response completed")
                    self.response_in_progress = False
                    # Safety flush
                    self._flush_audio_accum(force=True)
                    socketio.emit('response_complete', {}, room=self.session_id)
                
                elif event_type == "response.created":
                    print("Response generation started")
                    self.response_in_progress = True
                    socketio.emit('response_started', {}, room=self.session_id)
                        
                elif event_type == "input_audio_buffer.speech_started":
                    socketio.emit('speech_started', {}, room=self.session_id)
                    
                elif event_type == "input_audio_buffer.speech_stopped":
                    socketio.emit('speech_stopped', {}, room=self.session_id)
                    
                elif event_type == "error":
                    error_details = event.get("error", {})
                    socketio.emit('api_error', {'error': error_details}, room=self.session_id)
                    
            except Exception as e:
                print(f"Error in response listener: {e}")
                time.sleep(0.1)

    def _accumulate_or_emit_delta(self, b64_chunk: str):
        """Aggregate small audio delta chunks into larger packets to reduce client-side gaps.

        Strategy:
          - Collect sequential deltas until we approximate target duration (AGG_TARGET_MS) or reach byte cap.
          - If first chunk is small and no follow-up within AGG_MAX_WAIT_MS, flush early (low-latency bias).
          - On large single chunk, emit immediately.
        """
        # Decode length (avoid keeping decoded data until flush for memory efficiency)
        try:
            decoded = base64.b64decode(b64_chunk)
        except Exception:
            # If decode fails, emit original to avoid loss
            socketio.emit('audio_chunk', {'audio': b64_chunk}, room=self.session_id)
            return

        decoded_len = len(decoded)
        # Heuristic: if chunk already large enough (~>90ms), emit directly to reduce latency
        bytes_per_ms = (self._assumed_sample_rate * 2) / 1000  # 2 bytes per sample
        est_ms = decoded_len / bytes_per_ms
        now = time.time()

        if est_ms >= self.AGG_TARGET_MS * 0.75 and not self._audio_delta_accum:
            # Emit directly
            socketio.emit('audio_chunk', {'audio': b64_chunk}, room=self.session_id)
            return

        # Accumulate
        self._audio_delta_accum.append(b64_chunk)
        self._audio_accum_bytes += decoded_len
        if self._audio_first_delta_time is None:
            self._audio_first_delta_time = now

        accumulated_ms = self._audio_accum_bytes / bytes_per_ms
        waited_ms = (now - self._audio_first_delta_time) * 1000 if self._audio_first_delta_time else 0

        # Flush conditions
        if (accumulated_ms >= self.AGG_TARGET_MS) or (self._audio_accum_bytes >= self.AGG_MAX_BYTES) or (waited_ms >= self.AGG_MAX_WAIT_MS):
            self._flush_audio_accum()

    def _flush_audio_accum(self, force: bool = False):
        if not self._audio_delta_accum:
            return
        if not force and len(self._audio_delta_accum) == 1:
            # Single small chunk; if not forcing and not meeting criteria, let accumulate
            return
        try:
            raw = b''.join(base64.b64decode(c) for c in self._audio_delta_accum)
            merged_b64 = base64.b64encode(raw).decode('ascii')
            socketio.emit('audio_chunk', {'audio': merged_b64}, room=self.session_id)
        except Exception as e:
            print(f"Failed merging audio deltas: {e}. Emitting individually.")
            for c in self._audio_delta_accum:
                socketio.emit('audio_chunk', {'audio': c}, room=self.session_id)
        finally:
            self._audio_delta_accum.clear()
            self._audio_accum_bytes = 0
            self._audio_first_delta_time = None
    
    def send_audio(self, audio_data: str):
        """Send audio data to Voice Live API"""
        if self.connection and self.is_active:
            try:
                print(f"Sending audio data, length: {len(audio_data)}")
                param = {
                    "type": "input_audio_buffer.append", 
                    "audio": audio_data, 
                    "event_id": ""
                }
                self.connection.send(json.dumps(param))
                print("Audio data sent successfully")
            except Exception as e:
                print(f"Error sending audio: {e}")
    
    def trigger_response(self):
        """Trigger a response from the AI agent"""
        if self.connection and self.is_active and not self.response_in_progress:
            try:
                print("Triggering response generation")
                param = {
                    "type": "response.create",
                    "response": {
                        "modalities": ["text", "audio"],
                        "instructions": "Please respond to the user's input."
                    }
                }
                self.connection.send(json.dumps(param))
                print("Response trigger sent successfully")
            except Exception as e:
                print(f"Error triggering response: {e}")
        elif self.response_in_progress:
            print("Skipping response trigger - response already in progress")
        else:
            print("Cannot trigger response - connection not ready")
    
    def stop_session(self):
        """Stop the Voice Live session"""
        self.is_active = False
        self.response_in_progress = False
        if self.connection:
            self.connection.close()
        if self.audio_player:
            self.audio_player.terminate()
    
    def pause_session(self):
        """Pause the session without triggering responses"""
        print(f"Pausing session {self.session_id}")
        # Don't trigger any new responses when paused
        pass
    
    def resume_session(self):
        """Resume the session"""
        print(f"Resuming session {self.session_id}")
        # Session can now accept new inputs again
        pass

@app.route('/')
def index():
    """Serve the main web page"""
    return render_template('voice_live_web.html')

@socketio.on('connect')
def handle_connect():
    """Handle client connection"""
    session_id = request.sid
    print(f"Client connected: {session_id}")

@socketio.on('disconnect')
def handle_disconnect():
    """Handle client disconnection"""
    session_id = request.sid
    print(f"Client disconnected: {session_id}")
    if session_id in active_connections:
        active_connections[session_id].stop_session()
        del active_connections[session_id]

@socketio.on('start_voice_session')
def handle_start_session():
    """Start a new Voice Live session"""
    session_id = request.sid
    if session_id not in active_connections:
        voice_session = WebVoiceLiveSession(session_id)
        active_connections[session_id] = voice_session
        voice_session.start_session()

@socketio.on('audio_data')
def handle_audio_data(data):
    """Handle incoming audio data from client"""
    session_id = request.sid
    print(f"Received audio data from session {session_id}")
    if session_id in active_connections:
        audio_data = data.get('audio')
        if audio_data:
            print(f"Audio data length: {len(audio_data)}")
            active_connections[session_id].send_audio(audio_data)
        else:
            print("No audio data in received message")
    else:
        print(f"No active connection found for session {session_id}")

@socketio.on('trigger_response')
def handle_trigger_response():
    """Trigger AI agent response"""
    session_id = request.sid
    print(f"Triggering response for session {session_id}")
    if session_id in active_connections:
        active_connections[session_id].trigger_response()
    else:
        print(f"No active connection found for session {session_id}")

@socketio.on('pause_session')
def handle_pause_session():
    """Pause the session"""
    session_id = request.sid
    print(f"Pausing session {session_id}")
    if session_id in active_connections:
        active_connections[session_id].pause_session()
        emit('session_paused', {'status': 'success'})

@socketio.on('resume_session')
def handle_resume_session():
    """Resume the session"""
    session_id = request.sid
    print(f"Resuming session {session_id}")
    if session_id in active_connections:
        active_connections[session_id].resume_session()
        emit('session_resumed', {'status': 'success'})

@socketio.on('stop_voice_session')
def handle_stop_session():
    """Stop the Voice Live session"""
    session_id = request.sid
    if session_id in active_connections:
        active_connections[session_id].stop_session()
        del active_connections[session_id]
        emit('session_stopped', {'status': 'success'})

def check_dependencies():
    """Check if all required dependencies are available"""
    missing_deps = []
    
    # Check websocket-client
    if ws_client is None:
        missing_deps.append("websocket-client")
    
    # Check Azure libraries
    try:
        from azure.identity import DefaultAzureCredential
    except ImportError:
        missing_deps.append("azure-identity")
    
    # Check Flask libraries
    try:
        from flask import Flask
        from flask_socketio import SocketIO
    except ImportError:
        missing_deps.append("flask or flask-socketio")
    
    # Check voice_live_web module
    try:
        from voice_live_web import AzureVoiceLive, VoiceLiveConnection, AudioPlayerAsync
    except ImportError:
        missing_deps.append("voice_live_web module (check if voice_live_web.py exists)")
    
    if missing_deps:
        print("ERROR: Missing dependencies:")
        for dep in missing_deps:
            print(f"  - {dep}")
        print("\nPlease install missing packages using:")
        print("  pip install websocket-client azure-identity flask flask-socketio")
        return False
    
    return True

if __name__ == '__main__':
    # Check dependencies first
    if not check_dependencies():
        print("Please install missing dependencies before running the server.")
        exit(1)
    
    # Create templates directory if it doesn't exist
    os.makedirs('templates', exist_ok=True)
    
    # Set up logging
    logging.basicConfig(level=logging.INFO)
    
    print("Starting Voice Live Web Server...")
    print("Open your browser to http://localhost:5000")
    
    socketio.run(app, host='0.0.0.0', port=5000, debug=True)
```

- Now save the file as app.py
- run the app.py

```
python app.py
```

- Open the browser and go to http://localhost:5000
- You should see the web page

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/voiceagentex-1.jpg 'RagChat')

- Start the session

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/voiceagentex-2.jpg 'RagChat')

- Click the microphone and start speaking

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/voiceagentex-3.jpg 'RagChat')

- Start talking

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/voiceagentex-4.jpg 'RagChat')

- now wait for it respond

![info](https://github.com/balakreshnan/Samples2025/blob/main/AIFoundry/images/voiceagentex-5.jpg 'RagChat')

## Conclusion

- We have successfully created a voice enabled Azure AI Foundry agent using GPT realtime models
- This UI can be used to connect to any Azure AI Foundry agent
- Make sure feature is supported in the region based on availability
- Idea here is to build the agent separately and use this UI to connect to it.