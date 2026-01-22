# Events

> **Reference**: https://google.github.io/adk-docs/events/

## Overview

Events are the **fundamental units of information flow** within the Agent Development Kit. They represent every significant occurrence during agent interactions - from user input through final responses.

An `Event` is an **immutable record** that extends `LlmResponse` with ADK-specific metadata, enabling comprehensive tracking, debugging, and control flow management.

---

## Event Structure

### Core Fields

| Field | Type | Description |
|-------|------|-------------|
| `id` | `str` | Unique identifier for this event |
| `invocation_id` | `str` | Groups related events within one interaction cycle |
| `author` | `str` | Who created the event (`"user"` or agent name) |
| `content` | `Content` | The actual payload (text, function calls, etc.) |
| `partial` | `bool` | `True` if streaming chunk (incomplete) |
| `timestamp` | `datetime` | When the event was created |
| `actions` | `EventActions` | Control signals and side effects |
| `branch` | `str` | Execution path tracking in hierarchies |

### Content Structure

```python
# Text content
event.content.parts[0].text  # "Hello, how can I help?"

# Function call
event.content.parts[0].function_call  # {"name": "get_weather", "args": {...}}

# Function response
event.content.parts[0].function_response  # {"name": "get_weather", "response": {...}}
```

---

## Event Types

Events are classified by their `author` and `content`:

### By Author

| Author | Description |
|--------|-------------|
| `"user"` | Direct end-user input |
| Agent name (e.g., `"WeatherAgent"`) | Agent output or actions |

### By Content Type

| Type | How to Identify | Description |
|------|-----------------|-------------|
| **Text Message** | `event.content.parts[0].text` exists | Conversational content |
| **Tool Call Request** | `event.get_function_calls()` returns list | LLM requesting tool execution |
| **Tool Result** | `event.get_function_responses()` returns list | Tool execution response |
| **Streaming Chunk** | `event.partial == True` | Incomplete output during streaming |
| **State Update** | `event.actions.state_delta` exists | Session state modification |
| **Control Signal** | `event.actions.transfer_to_agent` or `escalate` | Flow control directive |

### Identifying Events in Code

```python
async for event in runner.run_async(...):
    # Check author
    if event.author == "user":
        print("User input")
    else:
        print(f"From agent: {event.author}")

    # Check for function calls
    func_calls = event.get_function_calls()
    if func_calls:
        for call in func_calls:
            print(f"Tool call: {call['name']}({call['args']})")

    # Check for function responses
    func_responses = event.get_function_responses()
    if func_responses:
        for resp in func_responses:
            print(f"Tool result: {resp['name']} -> {resp['response']}")

    # Check for streaming
    if event.partial:
        print(f"Streaming: {event.content.parts[0].text}", end="")
    else:
        print(f"Complete: {event.content.parts[0].text}")
```

---

## EventActions

`EventActions` contains signals for **side effects** and **control flow**.

### Fields

| Field | Type | Description |
|-------|------|-------------|
| `state_delta` | `dict` | Key-value pairs to merge into session state |
| `artifact_delta` | `dict` | `{filename: version}` for saved artifacts |
| `transfer_to_agent` | `str` | Agent name to transfer control to |
| `escalate` | `bool` | Signal to terminate loop iteration |
| `skip_summarization` | `bool` | Prevent LLM from summarizing tool results |
| `long_running_tool_ids` | `list` | IDs of background tool executions |

### state_delta

Updates session state through the event system:

```python
from google.adk.events import Event, EventActions

# Create state update event
state_changes = {
    "task_status": "active",
    "user:login_count": 5,
    "temp:validation_needed": True,
}

event = Event(
    invocation_id="inv_123",
    author="system",
    actions=EventActions(state_delta=state_changes),
)

# Apply via session service
await session_service.append_event(session, event)
```

**Via Context (Recommended)**:

```python
from google.adk.tools import ToolContext

def my_tool(data: str, tool_context: ToolContext) -> dict:
    """Tool that updates state."""
    # This automatically creates state_delta in the event
    tool_context.state["processed_data"] = data
    tool_context.state["user:action_count"] = (
        tool_context.state.get("user:action_count", 0) + 1
    )
    return {"status": "success"}
```

### escalate

Signals early termination of a `LoopAgent`:

```python
from google.adk.agents import Agent, LoopAgent
from google.adk.events import Event, EventActions

# Method 1: Custom agent with escalation
class QualityChecker(Agent):
    async def run_async(self, ctx):
        quality = evaluate_quality(ctx.state.get("draft", ""))
        is_done = quality >= 0.9

        yield Event(
            author=self.name,
            actions=EventActions(escalate=is_done),
        )

# Method 2: Tool-based escalation
def check_and_escalate(tool_context: ToolContext) -> dict:
    """Check quality and escalate if done."""
    quality = tool_context.state.get("quality_score", 0)
    if quality >= 0.9:
        # Set escalate flag
        tool_context.actions.escalate = True
        return {"status": "complete", "quality": quality}
    return {"status": "continue", "quality": quality}

# LoopAgent setup
refinement_loop = LoopAgent(
    name="refinement",
    sub_agents=[writer_agent, quality_checker],
    max_iterations=5,  # Hard limit
)
# Loop terminates when:
# 1. max_iterations reached, OR
# 2. Any sub-agent yields Event with escalate=True
```

### transfer_to_agent

Directs control to a specific agent:

```python
# Typically handled automatically by LLM in multi-agent setups
# The LLM generates: transfer_to_agent(agent_name='target_agent')
# ADK AutoFlow intercepts and routes accordingly

# Manual transfer (advanced use case)
event = Event(
    author="coordinator",
    actions=EventActions(transfer_to_agent="specialist_agent"),
)
```

### skip_summarization

Prevents LLM from summarizing tool output:

```python
def return_raw_data(tool_context: ToolContext) -> dict:
    """Return data that should not be summarized."""
    tool_context.actions.skip_summarization = True
    return {"raw_data": large_dataset}
```

---

## Event Processing Flow

```
User Query
    |
    v
Runner.run_async()
    |
    v
SessionService adds query to history
    |
    v
Agent.run_async() called
    |
    v
Agent yields Event  <------+
    |                      |
    v                      |
Runner receives Event      |
    |                      |
    v                      |
SessionService processes:  |
  - Merge state_delta      |
  - Update artifacts       |
  - Assign event.id        |
  - Append to history      |
    |                      |
    v                      |
Event yielded to caller    |
    |                      |
    v                      |
More events? -------------+
    |
    v (no)
Final response returned
```

---

## Key Methods

### is_final_response()

Filters intermediate steps from user-facing output:

```python
async for event in runner.run_async(...):
    if event.is_final_response():
        # Display to user
        print(event.content.parts[0].text)
```

**Returns `True` when:**
- Tool result with `skip_summarization=True`
- Long-running tool call detected
- Complete message without function calls/responses and not streaming

### get_function_calls()

Returns list of function calls in this event:

```python
func_calls = event.get_function_calls()
# Returns: [{"name": "get_weather", "args": {"city": "Seoul"}}]
```

### get_function_responses()

Returns list of function responses in this event:

```python
func_responses = event.get_function_responses()
# Returns: [{"name": "get_weather", "response": {"temp": 25}}]
```

---

## Common Event Patterns

### User Input Event

```python
Event(
    author="user",
    content=Content(parts=[Part(text="What's the weather?")]),
)
```

### Agent Response Event

```python
Event(
    author="WeatherAgent",
    content=Content(parts=[Part(text="The weather is sunny, 25C")]),
    partial=False,
)
```

### Tool Call Event

```python
Event(
    author="WeatherAgent",
    content=Content(parts=[Part(
        function_call=FunctionCall(
            name="get_weather",
            args={"city": "Seoul"}
        )
    )]),
)
```

### Tool Result Event

```python
Event(
    author="WeatherAgent",
    content=Content(parts=[Part(
        function_response=FunctionResponse(
            name="get_weather",
            response={"temp": 25, "condition": "sunny"}
        )
    )]),
)
```

### State Update Event

```python
Event(
    author="system",
    actions=EventActions(
        state_delta={"user:visit_count": 5},
        artifact_delta={"report.pdf": 1}
    ),
)
```

---

## Streaming Events

Handle streaming responses with partial events:

```python
async for event in runner.run_async(...):
    if event.partial:
        # Streaming chunk - append to display
        text = event.content.parts[0].text if event.content else ""
        print(text, end="", flush=True)
    elif event.is_final_response():
        # Complete response
        print()  # New line after streaming
        print(f"Final: {event.content.parts[0].text}")
```

---

## Event History Access

Access event history through session:

```python
session = await session_service.get_session(
    app_name="my_app",
    user_id="user_123",
    session_id="session_456"
)

# Iterate through all events
for event in session.events:
    print(f"[{event.author}] {event.content.parts[0].text if event.content else 'N/A'}")

# Filter by invocation
current_invocation = [e for e in session.events if e.invocation_id == "inv_123"]
```

---

## Best Practices

### 1. Use is_final_response() for User Output

```python
# Good - Uses built-in filtering
async for event in runner.run_async(...):
    if event.is_final_response():
        display_to_user(event)

# Avoid - Manual filtering is error-prone
async for event in runner.run_async(...):
    if event.author != "user" and not event.partial:
        display_to_user(event)  # May include tool calls
```

### 2. Check Existence Before Accessing Nested Properties

```python
# Good - Safe access
text = ""
if event.content and event.content.parts:
    text = event.content.parts[0].text or ""

# Avoid - May raise AttributeError
text = event.content.parts[0].text
```

### 3. Use invocation_id for Correlation

```python
# Group related events
invocation_events = {}
async for event in runner.run_async(...):
    inv_id = event.invocation_id
    if inv_id not in invocation_events:
        invocation_events[inv_id] = []
    invocation_events[inv_id].append(event)
```

### 4. Leverage EventActions for State, Not Direct Modification

```python
# Good - Through EventActions
tool_context.state["key"] = value  # Auto-creates state_delta

# Good - Explicit EventActions
event = Event(actions=EventActions(state_delta={"key": value}))

# Avoid - Direct session modification (won't persist)
session.state["key"] = value
```

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Events Overview | https://google.github.io/adk-docs/events/ |
| State Management | https://google.github.io/adk-docs/sessions/state/ |
| Sessions | https://google.github.io/adk-docs/sessions/ |
| Runtime | https://google.github.io/adk-docs/runtime/ |
