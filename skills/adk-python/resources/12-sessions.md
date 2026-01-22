# Sessions

> **Official Docs**:
> - [Sessions Overview](https://google.github.io/adk-docs/sessions/)
> - [Session Object](https://google.github.io/adk-docs/sessions/session/)
> - [Rewind Sessions](https://google.github.io/adk-docs/sessions/rewind/)
> - [State Management](https://google.github.io/adk-docs/sessions/state/)

## Overview

A **Session** represents a single conversation thread between a user and an agent. It maintains:
- **Event history**: All messages, tool calls, and agent responses
- **State**: Key-value data persisted across interactions
- **Metadata**: User ID, session ID, timestamps

Sessions enable continuity in multi-turn conversations and state persistence across interactions.

---

## Session Service

SessionService manages the lifecycle of sessions (create, get, list, delete) and persists session data.

### Available Implementations

| Service | Use Case | Persistence |
|---------|----------|-------------|
| `InMemorySessionService` | Development/testing | Memory only (lost on restart) |
| `DatabaseSessionService` | Production | SQLite, PostgreSQL, etc. |
| `VertexAiSessionService` | Vertex AI deployment | Managed storage |

### InMemorySessionService

Simple in-memory storage for development and testing:

```python
from google.adk.sessions import InMemorySessionService

session_service = InMemorySessionService()
```

### DatabaseSessionService

Persistent storage using SQLAlchemy-compatible databases:

```python
from google.adk.sessions import DatabaseSessionService

# SQLite (local file)
session_service = DatabaseSessionService(
    db_url="sqlite:///sessions.db"
)

# PostgreSQL (production)
session_service = DatabaseSessionService(
    db_url="postgresql://user:pass@localhost/mydb"
)
```

### Using with Runner

```python
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService

runner = Runner(
    agent=my_agent,
    app_name="my_app",
    session_service=InMemorySessionService(),
)

# Run with user and session IDs
async for event in runner.run_async(
    user_id="user_123",
    session_id="session_456",
    new_message="Hello!"
):
    print(event)
```

---

## Session Operations

### Create Session

```python
from google.adk.sessions import Session

session = await session_service.create_session(
    app_name="my_app",
    user_id="user_123",
    session_id="session_456",  # Optional: auto-generated if not provided
    state={"initial_key": "initial_value"},  # Optional: initial state
)
```

### Get Session

```python
session = await session_service.get_session(
    app_name="my_app",
    user_id="user_123",
    session_id="session_456",
)

if session:
    print(f"Events: {len(session.events)}")
    print(f"State: {session.state}")
```

### List Sessions

```python
sessions = await session_service.list_sessions(
    app_name="my_app",
    user_id="user_123",
)

for session in sessions:
    print(f"Session: {session.id}")
```

### Delete Session

```python
await session_service.delete_session(
    app_name="my_app",
    user_id="user_123",
    session_id="session_456",
)
```

---

## Rewind Sessions

Rewind allows restoring a session to a previous state by replaying events up to a specific point.

> **Reference**: https://google.github.io/adk-docs/sessions/rewind/

### Concept

Each event in a session has an `event_id`. By specifying an event ID, you can "rewind" the session to that point in history, effectively undoing subsequent events.

### Usage

```python
# Get session with all events
session = await session_service.get_session(
    app_name="my_app",
    user_id="user_123",
    session_id="session_456",
)

# Find the event to rewind to
target_event_id = "event_abc123"

# Rewind session
rewound_session = await session_service.rewind_session(
    app_name="my_app",
    user_id="user_123",
    session_id="session_456",
    event_id=target_event_id,
)

# Session state and history are now as of target_event_id
```

### Use Cases

| Use Case | Description |
|----------|-------------|
| Undo mistakes | Restore to before an incorrect agent action |
| Branch conversations | Create alternative conversation paths |
| Retry with modifications | Go back and try a different approach |
| Debugging | Replay events to identify issues |

### Important Notes

- Rewinding creates a new branch from the specified event
- Events after the rewind point are discarded
- State changes after the rewind point are reverted
- The session continues from the rewound state

---

## State Management

State provides a key-value store for persisting data across interactions within a session.

> **Reference**: https://google.github.io/adk-docs/sessions/state/

### State Types

State keys can be prefixed to control scope and persistence:

| Type | Prefix | Scope | Persistence | Example |
|------|--------|-------|-------------|---------|
| Session | (none) | Current session only | Per service | `current_step` |
| User | `user:` | All sessions for user | Persistent | `user:language` |
| App | `app:` | All users and sessions | Persistent | `app:version` |
| Temp | `temp:` | Current invocation only | Never persisted | `temp:calc_result` |

### State Type Examples

```python
from google.adk.tools import ToolContext

def my_tool(query: str, tool_context: ToolContext) -> str:
    """A tool that demonstrates state types."""

    # Session state (current session only)
    tool_context.state["current_query"] = query
    tool_context.state["step"] = "processing"

    # User state (persists across sessions for this user)
    tool_context.state["user:preferred_format"] = "json"
    tool_context.state["user:query_count"] = (
        tool_context.state.get("user:query_count", 0) + 1
    )

    # App state (shared across all users and sessions)
    tool_context.state["app:total_queries"] = (
        tool_context.state.get("app:total_queries", 0) + 1
    )

    # Temp state (current invocation only, not persisted)
    tool_context.state["temp:intermediate_result"] = "processing..."

    return f"Processed: {query}"
```

### State Scope Diagram

```
                    ┌─────────────────────────────────────┐
                    │           App State (app:)          │
                    │    Shared across all users/sessions │
                    ├─────────────────────────────────────┤
         ┌──────────┴──────────┐       ┌──────────┴──────────┐
         │   User A State      │       │   User B State      │
         │     (user:)         │       │     (user:)         │
         ├─────────────────────┤       ├─────────────────────┤
    ┌────┴────┐    ┌────┴────┐  ┌────┴────┐    ┌────┴────┐
    │Session 1│    │Session 2│  │Session 1│    │Session 2│
    │ (none)  │    │ (none)  │  │ (none)  │    │ (none)  │
    └─────────┘    └─────────┘  └─────────┘    └─────────┘
         │              │            │              │
    [temp: state is per-invocation within each session]
```

---

## Key Templating

Inject state values directly into agent instructions using template syntax.

### Basic Syntax

```python
from google.adk.agents import Agent

agent = Agent(
    name="personalized_agent",
    model="gemini-2.5-flash",
    instruction="You are helping {user:name}. Their preferred language is {user:language}.",
)

# When session.state["user:name"] = "Alice" and session.state["user:language"] = "English"
# LLM receives: "You are helping Alice. Their preferred language is English."
```

### Template Syntax Reference

| Syntax | Description | Behavior if Missing |
|--------|-------------|---------------------|
| `{var}` | Required variable | Error thrown |
| `{var?}` | Optional variable | Empty string substituted |
| `{artifact.name}` | Artifact text content | Error thrown |

### Examples

```python
# Required variable (throws error if missing)
instruction = "Process the following topic: {topic}"

# Optional variable (no error if missing)
instruction = "Previous context: {previous_result?}\nNew query: {current_query}"

# Multiple variables
instruction = """
User: {user:name}
Language: {user:language?}
Task: {current_task}
"""

# Artifact reference
instruction = "Review this document: {artifact.uploaded_doc}"
```

### Using with Workflow Agents

```python
from google.adk.agents import Agent, SequentialAgent

# Step 1: Generate draft
writer = Agent(
    name="writer",
    model="gemini-2.5-flash",
    instruction="Write a draft about the given topic.",
    output_key="draft",  # Saves response to state["draft"]
)

# Step 2: Review using previous output
reviewer = Agent(
    name="reviewer",
    model="gemini-2.5-flash",
    instruction="Review this draft: {draft}",  # References state["draft"]
    output_key="review",
)

# Step 3: Final edit using both outputs
editor = Agent(
    name="editor",
    model="gemini-2.5-flash",
    instruction="Edit the draft ({draft}) based on this review: {review}",
    output_key="final",
)

pipeline = SequentialAgent(
    name="writing_pipeline",
    sub_agents=[writer, reviewer, editor],
)
```

### Escaping Literal Braces

When you need literal braces in output (e.g., JSON), use an `InstructionProvider` function:

```python
def get_instruction(context):
    """Return instruction with literal braces."""
    topic = context.state.get("topic", "default")
    return f'Output JSON format: {{"topic": "{topic}", "status": "complete"}}'

agent = Agent(
    name="json_agent",
    model="gemini-2.5-flash",
    instruction=get_instruction,  # Pass function, not string
)
```

---

## output_key

The `output_key` parameter automatically saves an agent's final response to session state.

### Basic Usage

```python
agent = Agent(
    name="summarizer",
    model="gemini-2.5-flash",
    instruction="Summarize the input text.",
    output_key="summary",  # Response saved to state["summary"]
)

# After execution:
# session.state["summary"] = "The agent's summarized response..."
```

### With State Prefixes

```python
# Save to user state (persists across sessions)
agent = Agent(
    name="preference_collector",
    instruction="Ask the user about their preferences.",
    output_key="user:preferences",
)

# Save to app state (shared globally)
agent = Agent(
    name="config_generator",
    instruction="Generate default configuration.",
    output_key="app:default_config",
)
```

### Chaining Agents with output_key

```python
from google.adk.agents import SequentialAgent

research_agent = Agent(
    name="researcher",
    instruction="Research the topic thoroughly.",
    output_key="research_findings",
)

analysis_agent = Agent(
    name="analyst",
    instruction="Analyze these findings: {research_findings}",
    output_key="analysis",
)

report_agent = Agent(
    name="reporter",
    instruction="Write a report based on: {analysis}",
    output_key="final_report",
)

pipeline = SequentialAgent(
    name="research_pipeline",
    sub_agents=[research_agent, analysis_agent, report_agent],
)

# After execution:
# state["research_findings"] = "..."
# state["analysis"] = "..."
# state["final_report"] = "..."
```

---

## Best Practices

### State Management

| Practice | Description |
|----------|-------------|
| Use appropriate prefixes | `user:` for user-specific, `app:` for global, `temp:` for temporary |
| Keep values serializable | Use strings, numbers, booleans, lists, dicts only |
| Minimize state size | Store only essential data; avoid large objects |
| Use clear key names | `user:language` not `ul`, `cart:item_count` not `cnt` |

### Session Management

| Practice | Description |
|----------|-------------|
| Use consistent IDs | Same user_id across sessions for user state continuity |
| Clean up old sessions | Implement session cleanup/expiration for production |
| Choose appropriate service | InMemory for dev, Database for production |
| Handle missing sessions | Check if session exists before operations |

### Example: Production Setup

```python
from google.adk.runners import Runner
from google.adk.sessions import DatabaseSessionService

# Production-ready session service
session_service = DatabaseSessionService(
    db_url="postgresql://user:pass@db-host/sessions"
)

runner = Runner(
    agent=my_agent,
    app_name="production_app",
    session_service=session_service,
)

# Ensure session exists or create new one
async def get_or_create_session(user_id: str, session_id: str):
    session = await session_service.get_session(
        app_name="production_app",
        user_id=user_id,
        session_id=session_id,
    )
    if not session:
        session = await session_service.create_session(
            app_name="production_app",
            user_id=user_id,
            session_id=session_id,
            state={"created_at": datetime.now().isoformat()},
        )
    return session
```

---

## Anti-Patterns

### Direct State Modification

```python
# BAD - Changes not tracked or persisted
session = await session_service.get_session(...)
session.state["key"] = "value"  # Not persisted!

# GOOD - Use context.state in tools/callbacks
def my_tool(tool_context: ToolContext) -> str:
    tool_context.state["key"] = "value"  # Properly tracked
    return "done"

# GOOD - Use EventActions for service-level changes
from google.adk.events import Event, EventActions

event = Event(
    invocation_id="inv_1",
    author="system",
    actions=EventActions(state_delta={"key": "value"}),
)
await session_service.append_event(session, event)
```

### Storing Non-Serializable Data

```python
# BAD - Custom objects cannot be serialized
class User:
    def __init__(self, name):
        self.name = name

tool_context.state["user"] = User("Alice")  # Will fail

# GOOD - Use dictionaries
tool_context.state["user"] = {"name": "Alice"}
```

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Sessions Overview | https://google.github.io/adk-docs/sessions/ |
| Session Object | https://google.github.io/adk-docs/sessions/session/ |
| Rewind Sessions | https://google.github.io/adk-docs/sessions/rewind/ |
| State Management | https://google.github.io/adk-docs/sessions/state/ |
