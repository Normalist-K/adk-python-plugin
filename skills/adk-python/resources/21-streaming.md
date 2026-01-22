# Streaming (Bidi-streaming & Live API)

> **Reference**: https://google.github.io/adk-docs/streaming/

## Overview

ADK's bidi-streaming enables **low-latency bidirectional voice and video interaction** using Google's Gemini Live API. It supports natural, human-like voice conversations with the ability to interrupt agent responses mid-stream.

### Key Features

| Feature | Description |
|---------|-------------|
| **Real-time Communication** | Two-way streaming where both user and AI can speak simultaneously |
| **Multimodal Input** | Text, audio, and video through a single persistent connection |
| **Interruption Handling** | Users can interrupt agent responses with voice commands |
| **Voice Activity Detection** | Automatic speech start/stop detection |
| **Session Resumption** | Transparent reconnection for long sessions |

### Architecture

```
User Input (text/audio/video)
         ↓
   LiveRequestQueue ←── Upstream messages
         ↓
   Runner.run_live() ←── Event loop
         ↓
   Agent + Tools ←── Process & respond
         ↓
   Events (yield) ←── Downstream events
         ↓
   User Output (text/audio)
```

### Core Components

| Component | Role |
|-----------|------|
| **LiveRequestQueue** | Manages upstream messages (text, audio, video, control signals) |
| **run_live()** | Async generator yielding downstream events |
| **RunConfig** | Controls streaming behavior, modalities, session settings |

---

## Development Guide

### Part 1: Introduction to Streaming

> **Reference**: https://google.github.io/adk-docs/streaming/dev-guide/part1/

#### Platform Options

| Platform | Access | Use Case |
|----------|--------|----------|
| **Gemini Live API** | Google AI Studio (API key) | Rapid prototyping |
| **Vertex AI Live API** | Google Cloud | Production deployments |

#### Application Lifecycle

```python
# Phase 1: Initialization (once at startup)
from google.adk.agents import Agent
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService

agent = Agent(name="voice_agent", model="gemini-live-2.5-flash")
session_service = InMemorySessionService()
runner = Runner(agent=agent, app_name="my_app", session_service=session_service)

# Phase 2: Session Setup (per connection)
from google.adk.runners import RunConfig
from google.adk.live import LiveRequestQueue

run_config = RunConfig(
    response_modalities=["AUDIO"],
    streaming_mode="BIDI",
)
live_queue = LiveRequestQueue()

# Phase 3: Active Streaming (concurrent send/receive)
async def handle_session():
    async for event in runner.run_live(
        user_id="user_123",
        session_id="session_456",
        live_request_queue=live_queue,
        run_config=run_config,
    ):
        # Process events
        pass

# Phase 4: Termination
await live_queue.close()
```

---

### Part 2: Sending Messages

> **Reference**: https://google.github.io/adk-docs/streaming/dev-guide/part2/

#### LiveRequest Fields

| Field | Purpose | Exclusive |
|-------|---------|-----------|
| `content` | Text messages, structured data | Yes (with blob) |
| `blob` | Audio, video, binary streams | Yes (with content) |
| `activity_start` | Manual turn start signal | No |
| `activity_end` | Manual turn end signal | No |
| `close` | Graceful session termination | No |

#### Sending Messages

```python
from google.adk.live import LiveRequestQueue
from google.genai.types import Content, Part, Blob

live_queue = LiveRequestQueue()

# Text message (triggers immediate response)
await live_queue.send_content(
    content=Content(parts=[Part(text="Hello, how are you?")])
)

# Real-time audio streaming
await live_queue.send_realtime(
    blob=Blob(data=audio_bytes, mime_type="audio/pcm")
)

# Manual activity signals (when VAD is disabled)
await live_queue.send_activity_start()
await live_queue.send_activity_end()

# Graceful termination
await live_queue.close()
```

#### Best Practices

1. Create `LiveRequestQueue` within async context for correct event loop binding
2. Always call `close()` in finally blocks to prevent zombie sessions
3. Use convenience methods (`send_content`, `send_realtime`) over manual `LiveRequest` construction
4. Messages maintain FIFO ordering

---

### Part 3: Event Handling

> **Reference**: https://google.github.io/adk-docs/streaming/dev-guide/part3/

#### Event Types

| Event Type | Description | Persistence |
|------------|-------------|-------------|
| **Text** | Model text responses | Partial: No, Complete: Yes |
| **Audio (inline)** | Raw PCM bytes, real-time | No |
| **Audio (file)** | Aggregated artifact files | Yes (if `save_live_blob=True`) |
| **Transcription** | Speech-to-text results | Final only |
| **Usage Metadata** | Token counts | Yes |
| **Tool Calls** | Function execution events | Yes |
| **Errors** | Error codes and messages | - |

#### Processing Events

```python
async for event in runner.run_live(...):
    # Text response
    if event.content and event.content.parts:
        for part in event.content.parts:
            if part.text:
                if event.partial:
                    print(part.text, end="", flush=True)  # Streaming
                else:
                    print(part.text)  # Complete response

    # Audio response
    if event.content and event.content.parts:
        for part in event.content.parts:
            if part.inline_data:  # Raw audio bytes
                play_audio(part.inline_data.data)

    # Transcriptions
    if event.input_transcription:
        print(f"User said: {event.input_transcription}")
    if event.output_transcription:
        print(f"Agent said: {event.output_transcription}")

    # Turn completion
    if event.turn_complete:
        print("--- Turn complete ---")

    # Interruption
    if event.interrupted:
        print("--- User interrupted ---")

    # Usage metadata
    if event.usage_metadata:
        print(f"Tokens: {event.usage_metadata.total_token_count}")

    # Errors
    if event.error_code:
        print(f"Error: {event.error_code} - {event.error_message}")
```

#### Event Flags

| Flag | Description |
|------|-------------|
| `partial` | True for incremental chunks, False for complete text |
| `turn_complete` | Model finished its response |
| `interrupted` | User interrupted mid-response |

#### Serialization for Network

```python
# Convert event to JSON for WebSocket transmission
event_json = event.model_dump_json(exclude_none=True, by_alias=True)
await websocket.send_text(event_json)

# Optimization: Send audio as binary frames (75% bandwidth reduction)
if part.inline_data:
    await websocket.send_bytes(part.inline_data.data)
```

---

### Part 4: Run Configuration

> **Reference**: https://google.github.io/adk-docs/streaming/dev-guide/part4/

#### RunConfig Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `response_modalities` | List[str] | Output format: `["TEXT"]` or `["AUDIO"]` |
| `streaming_mode` | str | `"BIDI"` (WebSocket) or `"SSE"` (HTTP) |
| `session_resumption` | bool | Enable automatic reconnection |
| `context_window_compression` | bool | Enable unlimited session duration |
| `max_llm_calls` | int | Limit total LLM calls per session |
| `save_live_blob` | bool | Persist audio/video streams |
| `custom_metadata` | dict | Attach metadata to events |
| `support_cfc` | bool | Enable compositional function calling |

#### Configuration Example

```python
from google.adk.runners import RunConfig
from google.genai import types as genai_types

# Voice configuration
voice_config = genai_types.VoiceConfig(
    prebuilt_voice_config=genai_types.PrebuiltVoiceConfigDict(
        voice_name="Aoede"
    )
)

run_config = RunConfig(
    response_modalities=["AUDIO"],
    streaming_mode="BIDI",
    speech_config=genai_types.SpeechConfig(voice_config=voice_config),
    session_resumption=True,
    context_window_compression=True,
    max_llm_calls=100,
    save_live_blob=True,
)
```

#### Streaming Modes

| Mode | Protocol | Use Case |
|------|----------|----------|
| `BIDI` | WebSocket | Real-time audio/video via Live API |
| `SSE` | HTTP | Text streaming via standard Gemini API |

#### Production Limits

| Platform | Concurrent Sessions | Duration Limit |
|----------|---------------------|----------------|
| Gemini Live API | 50-1,000 (tier-based) | 15 min (audio), 2 min (video) |
| Vertex AI Live API | Up to 1,000/project | 10 min (without compression) |

**Note**: Enable `context_window_compression` to bypass duration limits.

---

### Part 5: Audio, Images, and Video

> **Reference**: https://google.github.io/adk-docs/streaming/dev-guide/part5/

#### Audio Specifications

| Direction | Format | Sample Rate | Channels |
|-----------|--------|-------------|----------|
| **Input** | 16-bit PCM (signed int) | 16,000 Hz | Mono |
| **Output** | 16-bit PCM | 24,000 Hz | Mono |

#### Audio Chunk Sizes

| Latency | Chunk Size | Bytes |
|---------|------------|-------|
| Ultra-low | 10-20ms | 320-640 |
| Balanced (recommended) | 50-100ms | 1,600-3,200 |
| Lower overhead | 100-200ms | 3,200-6,400 |

#### Sending Audio

```python
# Convert float32 to 16-bit PCM
def float32_to_pcm16(float_audio):
    return (float_audio * 0x7fff).astype(np.int16).tobytes()

# Stream audio chunks
audio_chunk = float32_to_pcm16(microphone_data)
await live_queue.send_realtime(
    blob=Blob(data=audio_chunk, mime_type="audio/pcm")
)
```

#### Image/Video Specifications

| Property | Specification |
|----------|---------------|
| **Format** | JPEG |
| **Frame Rate** | Max 1 FPS |
| **Resolution** | 768x768 pixels (recommended) |

**Limitation**: Not suitable for real-time motion tracking or fast-moving subjects.

#### Sending Video Frames

```python
# Capture and send frame
frame_jpeg = capture_frame_as_jpeg()  # JPEG bytes
await live_queue.send_realtime(
    blob=Blob(data=frame_jpeg, mime_type="image/jpeg")
)
```

#### Audio Model Architectures

| Type | Model Example | Features |
|------|--------------|----------|
| **Native Audio** | `gemini-2.5-flash-native-audio-preview-*` | End-to-end audio, extended voices, affective dialog |
| **Half-Cascade** | `gemini-live-2.5-flash` | Native audio input + TTS output, faster text responses |

#### Voice Options

Half-cascade models support 8 prebuilt voices. Native audio models support extended voice library.

```python
# Available voices: Aoede, Charon, Fenrir, Kore, Puck, etc.
voice_config = genai_types.VoiceConfig(
    prebuilt_voice_config=genai_types.PrebuiltVoiceConfigDict(
        voice_name="Puck"
    )
)
```

---

## Streaming Tools

> **Reference**: https://google.github.io/adk-docs/streaming/streaming-tools/

Streaming tools send intermediate results back to agents in real-time.

### Requirements

1. Must be an `async` function
2. Must return `AsyncGenerator` type
3. First type parameter specifies yielded data type

### Simple Streaming Tool

```python
from typing import AsyncGenerator

async def monitor_stock_price(stock_symbol: str) -> AsyncGenerator[str, None]:
    """Monitor stock price and yield alerts."""
    while True:
        price = await get_current_price(stock_symbol)
        if price > threshold:
            yield f"Alert: {stock_symbol} is now ${price}"
        await asyncio.sleep(5)
```

### Video Streaming Tool

```python
from google.adk.live import LiveRequestQueue
from typing import AsyncGenerator

async def monitor_video_stream(
    input_stream: LiveRequestQueue  # Reserved parameter for video
) -> AsyncGenerator[str, None]:
    """Analyze video frames and yield alerts."""
    async for frame in input_stream:
        if detect_anomaly(frame):
            yield "Anomaly detected in video feed"
```

### Stop Streaming Tool

```python
from google.adk.tools import FunctionTool

def stop_streaming() -> dict:
    """Stop all streaming operations."""
    return {"status": "stopped"}

# Register both tools
agent = Agent(
    name="monitor_agent",
    model="gemini-live-2.5-flash",
    tools=[monitor_stock_price, FunctionTool(stop_streaming)],
)
```

---

## Configuration Options

> **Reference**: https://google.github.io/adk-docs/streaming/configuration/

### Complete RunConfig Reference

```python
from google.adk.runners import RunConfig
from google.genai import types as genai_types

run_config = RunConfig(
    # Response format (choose one, cannot switch mid-session)
    response_modalities=["AUDIO"],  # or ["TEXT"]

    # Streaming protocol
    streaming_mode="BIDI",  # "BIDI" or "SSE"

    # Voice settings
    speech_config=genai_types.SpeechConfig(
        voice_config=genai_types.VoiceConfig(
            prebuilt_voice_config=genai_types.PrebuiltVoiceConfigDict(
                voice_name="Aoede"
            )
        )
    ),

    # Session management
    session_resumption=True,  # Auto-reconnect on timeout
    context_window_compression=True,  # Unlimited duration

    # Safety limits
    max_llm_calls=100,

    # Persistence
    save_live_blob=True,  # Save audio/video to artifacts

    # Metadata
    custom_metadata={"user_tier": "premium"},

    # Advanced
    support_cfc=True,  # Compositional function calling
)
```

### Session Resumption

ADK automatically handles Gemini's ~10-minute connection timeouts:

```python
run_config = RunConfig(
    session_resumption=True,  # Enabled
)
```

When enabled, ADK transparently reconnects using cached resumption handles.

### Context Window Compression

For sessions exceeding duration/token limits:

```python
run_config = RunConfig(
    context_window_compression=True,
)
```

Compresses older conversation history when token thresholds are reached.

---

## Best Practices

### Production Checklist

| Area | Recommendation |
|------|----------------|
| **Session Management** | Enable `session_resumption` for reliability |
| **Duration Limits** | Enable `context_window_compression` for long sessions |
| **Resource Cleanup** | Always call `live_queue.close()` in finally blocks |
| **Error Handling** | Implement retry logic for transient errors |
| **Quota Management** | Monitor concurrent sessions, implement queueing if needed |
| **Audio Optimization** | Send binary frames over WebSocket (avoid base64) |

### Error Handling Strategy

| Error Type | Action |
|------------|--------|
| Content policy violations | Break, notify user |
| Token limits | Break, summarize and restart |
| Transient network issues | Continue with backoff |
| Rate limits | Continue with exponential backoff |

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Streaming Overview | https://google.github.io/adk-docs/streaming/ |
| Part 1: Introduction | https://google.github.io/adk-docs/streaming/dev-guide/part1/ |
| Part 2: Sending Messages | https://google.github.io/adk-docs/streaming/dev-guide/part2/ |
| Part 3: Event Handling | https://google.github.io/adk-docs/streaming/dev-guide/part3/ |
| Part 4: Run Configuration | https://google.github.io/adk-docs/streaming/dev-guide/part4/ |
| Part 5: Audio/Images/Video | https://google.github.io/adk-docs/streaming/dev-guide/part5/ |
| Streaming Tools | https://google.github.io/adk-docs/streaming/streaming-tools/ |
| Configuration | https://google.github.io/adk-docs/streaming/configuration/ |
