# Runtime

> **Reference**: https://google.github.io/adk-docs/runtime/

## Overview

ADK Runtime provides multiple ways to run and interact with agents: Web Interface for development, Command Line for testing, and API Server for production deployment.

---

## Web Interface

> **Reference**: https://google.github.io/adk-docs/runtime/web-interface/

The `adk web` command launches a browser-based development UI for testing and debugging agents.

### Basic Usage

```bash
# Single agent
adk web my_agent

# Multiple agents (recommended)
adk web backend/agents  # All agents in directory selectable via UI

# Custom port
adk web --port 9100 my_agent
```

### CLI Options

| Option | Default | Description |
|--------|---------|-------------|
| `--port` | `8000` | Server port |
| `--host` | `127.0.0.1` | Binding host |
| `--session_service_uri` | `.adk/session.db` | Session storage URI |
| `--log_level` | `INFO` | Logging level |
| `--reload_agents` | `False` | Auto-reload on file changes |

### Dev UI Features

| Feature | Description |
|---------|-------------|
| **Chat Interface** | Interactive conversation with agent |
| **Session Inspector** | View session state and history |
| **Event Viewer** | Real-time event stream |
| **Agent Selector** | Switch between multiple agents |
| **Visual Builder** | Create agents via drag-and-drop UI |

### Session Persistence

```bash
# Default: SQLite persistence
adk web my_agent  # Sessions saved to .adk/session.db

# In-memory (no persistence)
adk web my_agent --session_service_uri memory://
```

---

## Command Line

> **Reference**: https://google.github.io/adk-docs/runtime/command-line/

The `adk run` command runs agents in the terminal for quick testing.

### Basic Usage

```bash
# Interactive mode
adk run my_agent

# With initial message
adk run my_agent --message "Hello"

# Custom session
adk run my_agent --user_id user123 --session_id session456
```

### CLI Options

| Option | Default | Description |
|--------|---------|-------------|
| `--message, -m` | - | Initial message to send |
| `--user_id` | `default_user` | User identifier |
| `--session_id` | Auto-generated | Session identifier |
| `--session_service_uri` | `.adk/session.db` | Session storage URI |
| `--log_level` | `INFO` | Logging level |
| `--verbose, -v` | `False` | Enable DEBUG logging |

### Examples

```bash
# Quick test with message
adk run my_agent -m "What's the weather in Seoul?"

# Continue existing session
adk run my_agent --session_id my_session_123

# Debug mode
adk run my_agent --verbose
```

---

## API Server

> **Reference**: https://google.github.io/adk-docs/runtime/api-server/

The `adk api_server` command launches a FastAPI-based REST API server.

### Basic Usage

```bash
# Default (port 8000)
adk api_server backend/agents

# Custom configuration
adk api_server backend/agents \
  --host 0.0.0.0 \
  --port 9100 \
  --session_service_uri "sqlite+aiosqlite:///./sessions.db"
```

### CLI Options

| Option | Default | Description |
|--------|---------|-------------|
| `--host` | `127.0.0.1` | Binding host |
| `--port` | `8000` | Server port |
| `--session_service_uri` | `.adk/session.db` | Session storage URI |
| `--log_level` | `INFO` | Logging level |
| `--reload_agents` | `False` | Auto-reload on changes |
| `--allow_origins` | - | CORS allowed origins |
| `--trace_to_cloud` | `False` | Enable Cloud Trace |
| `--a2a` | `False` | Enable A2A endpoints |

### REST Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/apps/{app}/users/{user}/sessions/{session}` | POST | Create session |
| `/apps/{app}/users/{user}/sessions/{session}` | GET | Get session |
| `/run` | POST | Execute query (batch) |
| `/run_sse` | POST | Execute query (streaming) |
| `/docs` | GET | Swagger UI |

### Create Session

```bash
curl -X POST "http://localhost:8000/apps/my_agent/users/user123/sessions/session456" \
  -H "Content-Type: application/json" \
  -d '{"state": {"language": "en"}}'
```

### Execute Query (Batch)

```bash
curl -X POST "http://localhost:8000/run" \
  -H "Content-Type: application/json" \
  -d '{
    "appName": "my_agent",
    "userId": "user123",
    "sessionId": "session456",
    "newMessage": {
      "role": "user",
      "parts": [{"text": "Hello"}]
    }
  }'
```

### Execute Query (Streaming)

```bash
curl -X POST "http://localhost:8000/run_sse" \
  -H "Content-Type: application/json" \
  -d '{
    "appName": "my_agent",
    "userId": "user123",
    "sessionId": "session456",
    "newMessage": {
      "role": "user",
      "parts": [{"text": "Tell me a story"}]
    }
  }'
```

---

## Resume Agents

> **Reference**: https://google.github.io/adk-docs/runtime/resume/

Resume allows continuing interrupted agent workflows from the last checkpoint.

### Enable Resumability

```python
from google.adk.runners import Runner, RunConfig

config = RunConfig(
    support_cfc=True,  # Enable checkpoint support
)

runner = Runner(
    agent=my_agent,
    app_name="my_app",
    session_service=session_service,
)
```

### Resume Interrupted Workflow

```python
# Resume using invocation_id
async for event in runner.run_async(
    user_id="user123",
    session_id="session456",
    invocation_id="previous_invocation_id",  # Resume point
):
    print(event)
```

### Resume Behavior by Agent Type

| Agent Type | Resume Behavior |
|------------|-----------------|
| `SequentialAgent` | Continues from next unexecuted sub-agent |
| `LoopAgent` | Continues from last iteration and sub-agent |
| `ParallelAgent` | Executes only incomplete sub-agents |
| `LlmAgent` | Resumes from last tool call or response |

### Idempotent Tool Design

When using Resume, ensure tools are idempotent to prevent duplicate operations:

```python
from google.adk.tools import ToolContext

def purchase_item(item_id: str, tool_context: ToolContext) -> dict:
    """Purchase an item with idempotency check.

    Args:
        item_id: The item to purchase

    Returns:
        Purchase result
    """
    # Check if already purchased
    key = f"purchased:{item_id}"
    if tool_context.state.get(key):
        return {"status": "already_purchased"}

    # Execute purchase
    result = execute_purchase(item_id)

    # Mark as completed
    tool_context.state[key] = True
    return {"status": "success", "result": result}
```

---

## RunConfig

> **Reference**: https://google.github.io/adk-docs/runtime/runconfig/

RunConfig controls agent execution behavior.

### Basic Usage

```python
from google.adk.runners import Runner, RunConfig
from google.adk.sessions import InMemorySessionService

config = RunConfig(
    max_llm_calls=100,
    streaming_mode="SSE",
)

runner = Runner(
    agent=agent,
    app_name="my_app",
    session_service=InMemorySessionService(),
)

# Pass config when running
response = runner.run(
    user_id="user123",
    session_id="session456",
    new_message="Hello",
    run_config=config,
)
```

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `max_llm_calls` | 500 | Maximum LLM calls per run (prevents infinite loops) |
| `streaming_mode` | `NONE` | Output streaming mode |
| `speech_config` | - | Speech synthesis config (Live API) |
| `response_modalities` | - | Output format (AUDIO, TEXT) |
| `output_audio_transcription` | - | Audio transcription settings |
| `support_cfc` | False | Enable resume/checkpoint support |

### streaming_mode Options

| Mode | Description | Use Case |
|------|-------------|----------|
| `NONE` | Complete response after execution | Batch processing |
| `SSE` | Server-to-client streaming | Web apps, real-time UI |
| `BIDI` | Bidirectional real-time | Voice assistants, Live API |

### Prevent Infinite Loops

```python
config = RunConfig(
    max_llm_calls=50,  # Force stop after 50 LLM calls
)
```

### Speech Configuration (Live API)

```python
from google.adk.runners import RunConfig, SpeechConfig

config = RunConfig(
    streaming_mode="BIDI",
    speech_config=SpeechConfig(
        voice_config={"voice_name": "Kore"},
        language_code="en-US",
    ),
    response_modalities=["AUDIO"],
)
```

---

## Event Loop

> **Reference**: https://google.github.io/adk-docs/runtime/event-loop/

The Runner orchestrates agent execution through an event-driven loop.

### Core Components

| Component | Role |
|-----------|------|
| **Runner** | Central orchestrator, manages services |
| **Agent** | Core logic, yields events |
| **SessionService** | Session state and history management |
| **Event** | Communication between Runner and Agent |

### Execution Flow

```
User Query
    |
    v
Runner.run_async()
    |
    v
Add query to SessionService
    |
    v
Call Agent.run_async()
    |
    v
Agent yields Event
    |
    v
Runner processes Event (commit state)
    |
    v
Wait for next Event (loop)
    |
    v
Return final response
```

### Programmatic Usage

```python
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService

runner = Runner(
    agent=my_agent,
    app_name="my_app",
    session_service=InMemorySessionService(),
)

# Synchronous execution
response = runner.run(
    user_id="user123",
    session_id="session456",
    new_message="Hello",
)
print(response.text)

# Asynchronous execution (streaming)
async for event in runner.run_async(
    user_id="user123",
    session_id="session456",
    new_message="Hello",
):
    if event.is_final:
        print("Final:", event.text)
    elif event.partial:
        print(event.text, end="", flush=True)
```

### Event Types

| Event Type | Description |
|------------|-------------|
| `TextEvent` | Text response (partial or final) |
| `ToolCallEvent` | Tool invocation |
| `ToolResultEvent` | Tool execution result |
| `StateUpdateEvent` | State modification |
| `ErrorEvent` | Error occurred |

### Custom Event Processing

```python
async for event in runner.run_async(
    user_id="user123",
    session_id="session456",
    new_message="Hello",
):
    match event.type:
        case "text":
            if event.is_final:
                print(f"Assistant: {event.text}")
            else:
                print(event.text, end="")
        case "tool_call":
            print(f"Calling tool: {event.tool_name}")
        case "tool_result":
            print(f"Tool result: {event.result}")
        case "error":
            print(f"Error: {event.error}")
```

### SessionService Options

| Service | Persistence | Use Case |
|---------|-------------|----------|
| `InMemorySessionService` | No | Development, testing |
| `DatabaseSessionService` | Yes | Production (SQLite, PostgreSQL) |
| `VertexAiSessionService` | Yes | Google Cloud managed |

```python
from google.adk.sessions import DatabaseSessionService

# SQLite
session_service = DatabaseSessionService(
    db_url="sqlite+aiosqlite:///./sessions.db"
)

# PostgreSQL
session_service = DatabaseSessionService(
    db_url="postgresql+asyncpg://user:pass@host:5432/db"
)
```

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Runtime Overview | https://google.github.io/adk-docs/runtime/ |
| Web Interface | https://google.github.io/adk-docs/runtime/web-interface/ |
| Command Line | https://google.github.io/adk-docs/runtime/command-line/ |
| API Server | https://google.github.io/adk-docs/runtime/api-server/ |
| Resume | https://google.github.io/adk-docs/runtime/resume/ |
| RunConfig | https://google.github.io/adk-docs/runtime/runconfig/ |
| Event Loop | https://google.github.io/adk-docs/runtime/event-loop/ |
