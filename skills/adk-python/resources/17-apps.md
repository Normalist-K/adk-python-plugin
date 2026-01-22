# Apps

> **Official Docs**:
> - [Apps Overview](https://google.github.io/adk-docs/apps/)
> - [ADK Architecture](https://google.github.io/adk-docs/get-started/about/)

## Overview

The `App` class is a **top-level container** that manages the lifecycle, configuration, and state for agent workflows. It provides a formal boundary between operational infrastructure and individual agents' reasoning tasks.

| Concern | Without App | With App |
|---------|-------------|----------|
| Configuration | Scattered across agents | Centralized in App |
| Lifecycle | Manual management | Startup/shutdown hooks |
| State Scope | Implicit | Explicit (`app:*` prefix) |
| Deployment | Ad-hoc | Formal versioned unit |

---

## Technical Architecture

ADK follows a layered architecture where App sits at the top:

```
┌─────────────────────────────────────────────────────┐
│                       App                           │
│  (name, root_agent, plugins, caching config)        │
└─────────────────────────────┬───────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────┐
│                     Runner                          │
│  (execution engine, orchestrates agent flow)        │
└─────────────────────────────┬───────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────┐
│                    Session                          │
│  (conversation context, events, state)              │
└─────────────────────────────┬───────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────┐
│               Agent (root_agent)                    │
│  ├── LLM Agent (reasoning, tool selection)          │
│  ├── Workflow Agent (Sequential, Loop, Parallel)    │
│  └── sub_agents (delegation hierarchy)              │
└─────────────────────────────┬───────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────┐
│                     Tools                           │
│  (external APIs, RAG search, code execution)        │
└─────────────────────────────────────────────────────┘
```

### Core Components

| Component | Description |
|-----------|-------------|
| **App** | Top-level container for agent workflows |
| **Runner** | Execution engine that orchestrates agent interactions |
| **Session** | Individual conversation context with history (Events) and State |
| **Agent** | Fundamental execution unit (LLM or Workflow) |
| **Tools** | Extensions enabling interaction with external systems |
| **Events** | Basic communication units (user messages, agent replies, tool calls) |
| **State** | Short-term working memory within a session |

---

## App Class

### Constructor Parameters

```python
from google.adk.apps import App
from google.adk.agents import Agent

app = App(
    name="my_app",
    root_agent=my_agent,
    # Optional configurations
    plugins=[],
    context_cache_config=None,
    resumability_config=None,
)
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | `str` | Yes | Application identifier |
| `root_agent` | `Agent` | Yes | Primary controller agent |
| `plugins` | `list` | No | Plugin configurations |
| `context_cache_config` | `ContextCacheConfig` | No | Context caching settings |
| `resumability_config` | `ResumabilityConfig` | No | Agent resume configurations |

### Basic Example

```python
from google.adk.agents import Agent
from google.adk.apps import App

# Define root agent
root_agent = Agent(
    name="greeter_agent",
    model="gemini-2.5-flash",
    description="A friendly greeting agent",
    instruction="Reply with a warm greeting to the user.",
)

# Create App container
app = App(
    name="greeting_app",
    root_agent=root_agent,
)
```

---

## App Configuration Options

### Context Cache Config

Enables caching of context to improve performance for repeated queries:

```python
from google.adk.apps import App, ContextCacheConfig

app = App(
    name="cached_app",
    root_agent=root_agent,
    context_cache_config=ContextCacheConfig(
        enabled=True,
        ttl_seconds=3600,  # Cache TTL
    ),
)
```

### Events Compaction Config

Manages event history compaction for long-running sessions:

```python
from google.adk.apps import App

app = App(
    name="compacted_app",
    root_agent=root_agent,
    events_compaction_config={
        "enabled": True,
        "max_events": 100,
        "summarize": True,
    },
)
```

---

## App Lifecycle

### Lifecycle Phases

```
1. Initialization → 2. Startup → 3. Running → 4. Shutdown
```

| Phase | Description | Hook |
|-------|-------------|------|
| **Initialization** | App instance created, config validated | Constructor |
| **Startup** | Resources initialized (DB connections, caches) | `on_startup` |
| **Running** | Processing requests via Runner | `run_async` |
| **Shutdown** | Resources released, cleanup | `on_shutdown` |

### Lifecycle Hooks

```python
from google.adk.apps import App

class MyApp(App):
    async def on_startup(self):
        """Called when app starts - initialize resources"""
        self.db_connection = await create_db_connection()
        self.cache = await initialize_cache()
        print("App started, resources initialized")

    async def on_shutdown(self):
        """Called when app stops - cleanup resources"""
        await self.db_connection.close()
        await self.cache.clear()
        print("App shutdown, resources released")
```

---

## Running an App

### With InMemoryRunner

For local development and testing:

```python
from google.adk.apps import App
from google.adk.runners import InMemoryRunner
from google.adk.agents import Agent
from google.genai import types

# Create agent and app
root_agent = Agent(
    name="assistant",
    model="gemini-2.5-flash",
    instruction="You are a helpful assistant.",
)

app = App(name="assistant_app", root_agent=root_agent)

# Create runner
runner = InMemoryRunner(app=app, app_name=app.name)

# Run conversation
async def main():
    # Create session
    session = await runner.session_service.create_session(
        app_name=app.name,
        user_id="user123",
    )

    # Send message
    user_content = types.Content(
        role="user",
        parts=[types.Part(text="Hello!")]
    )

    async for event in runner.run_async(
        user_id="user123",
        session_id=session.id,
        new_message=user_content,
    ):
        if event.content and event.content.parts:
            print(event.content.parts[0].text)

# Execute
import asyncio
asyncio.run(main())
```

### With ADK CLI

The ADK CLI automatically creates an App wrapper for your root_agent:

```bash
# Dev UI (browser-based testing)
adk web backend/agents/my_agent

# CLI mode
adk run backend/agents/my_agent

# API Server
adk api_server backend/agents --port 9101
```

When using CLI, ensure your `agent.py` exports `root_agent`:

```python
# backend/agents/my_agent/agent.py
from google.adk.agents import Agent

root_agent = Agent(
    name="my_agent",
    model="gemini-2.5-flash",
    instruction="Your instructions here",
)
```

---

## App with Session Persistence

### Using SqliteSessionService

```python
from google.adk.apps import App
from google.adk.runners import Runner
from google.adk.sessions.sqlite_session_service import SqliteSessionService

# Create session service with persistence
session_service = SqliteSessionService(db_path="./sessions.db")

# Create runner with persistent sessions
runner = Runner(
    app=app,
    app_name=app.name,
    session_service=session_service,
)

# Sessions now persist across restarts
```

### Session Service Options

| Service | Persistence | Use Case |
|---------|-------------|----------|
| `InMemorySessionService` | None (lost on restart) | Local testing |
| `SqliteSessionService` | SQLite file | Development, small deployments |
| `DatabaseSessionService` | PostgreSQL/MySQL | Production |

---

## Multi-Agent App

```python
from google.adk.agents import Agent, SequentialAgent
from google.adk.apps import App

# Specialized agents
researcher = Agent(
    name="researcher",
    model="gemini-2.5-flash",
    instruction="Research the topic thoroughly.",
    output_key="research",
)

writer = Agent(
    name="writer",
    model="gemini-2.5-flash",
    instruction="Write based on research: {research}",
    output_key="article",
)

# Coordinator (root agent)
root_agent = SequentialAgent(
    name="content_pipeline",
    sub_agents=[researcher, writer],
)

# App with multi-agent workflow
app = App(
    name="content_app",
    root_agent=root_agent,
)
```

---

## State Management

### State Scopes

ADK provides three state scopes accessible via prefix:

| Prefix | Scope | Lifetime | Use Case |
|--------|-------|----------|----------|
| `app:*` | Application | App lifetime | Shared config, global counters |
| `user:*` | User | Across sessions | User preferences, profile |
| (none) | Session | Current session | Conversation context |

### Accessing State in Tools

```python
from google.adk.tools import ToolContext

def my_tool(query: str, tool_context: ToolContext) -> str:
    """Tool with state access."""
    # Session state
    session_count = tool_context.state.get("query_count", 0)
    tool_context.state["query_count"] = session_count + 1

    # App-level state
    app_config = tool_context.state.get("app:config", {})

    # User-level state
    user_prefs = tool_context.state.get("user:preferences", {})

    return f"Processed query #{session_count + 1}"
```

---

## Best Practices

### 1. Single Root Agent

Always define one root_agent that orchestrates others:

```python
# Good: Clear hierarchy
coordinator = Agent(
    name="coordinator",
    sub_agents=[agent_a, agent_b],
)
app = App(name="my_app", root_agent=coordinator)

# Bad: Multiple uncoordinated agents
# app = App(name="my_app", agents=[agent_a, agent_b])  # Not supported
```

### 2. Use Lifecycle Hooks for Resources

```python
class ProductionApp(App):
    async def on_startup(self):
        # Initialize expensive resources once
        self.vector_db = await init_vector_db()
        self.llm_cache = LRUCache(maxsize=1000)

    async def on_shutdown(self):
        # Cleanup properly
        await self.vector_db.close()
        self.llm_cache.clear()
```

### 3. Configure Session Persistence for Production

```python
# Development
runner = InMemoryRunner(app=app, app_name=app.name)

# Production
session_service = SqliteSessionService(db_path="./sessions.db")
runner = Runner(app=app, app_name=app.name, session_service=session_service)
```

### 4. Use Meaningful App Names

```python
# Good: Descriptive, versioned
app = App(name="customer_support_v2", root_agent=support_agent)

# Bad: Generic
app = App(name="app", root_agent=agent)
```

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Apps Overview | https://google.github.io/adk-docs/apps/ |
| ADK Architecture | https://google.github.io/adk-docs/get-started/about/ |
| Sessions & State | https://google.github.io/adk-docs/sessions/ |
| Runners | https://google.github.io/adk-docs/runtime/ |
| CLI Reference | https://google.github.io/adk-docs/runtime/cli/ |
