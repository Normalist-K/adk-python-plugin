# Memory Service

> **Official Docs**: https://google.github.io/adk-docs/sessions/memory/

## Overview

The `MemoryService` enables agents to access **long-term knowledge** across multiple sessions. Think of it as a searchable archive that persists beyond individual conversations.

| Concept | Analogy | Scope |
|---------|---------|-------|
| **Session / State** | Short-term memory during one chat | Single session |
| **MemoryService** | Searchable archive | Across all sessions |

---

## Memory Service Types

ADK provides two memory service implementations:

| Type | Use Case | Persistence | Search Method |
|------|----------|-------------|---------------|
| `InMemoryMemoryService` | Prototyping, local dev | None (lost on restart) | Basic keyword matching |
| `VertexAiMemoryBankService` | Production | Managed by Vertex AI | LLM-powered semantic search |

### InMemoryMemoryService

Simple in-memory storage for development and testing.

```python
from google.adk.memory import InMemoryMemoryService

memory_service = InMemoryMemoryService()
```

**Characteristics:**
- No configuration required
- Data lost on application restart
- Basic keyword matching for search
- Suitable for prototyping only

### VertexAiMemoryBankService

Production-ready memory service with semantic search capabilities.

```python
from google.adk.memory import VertexAiMemoryBankService

memory_service = VertexAiMemoryBankService(
    project="your-gcp-project-id",
    location="us-central1",
    agent_engine_id="your-agent-engine-id",
)
```

**Characteristics:**
- Uses LLM-powered extraction to consolidate meaningful information
- Enables semantic search across stored memories
- Persistent storage managed by Google Cloud
- Requires GCP project and Agent Engine instance

**Requirements:**
- Google Cloud Project
- Agent Engine instance
- Proper authentication (service account or ADC)

---

## Core API

### Adding Sessions to Memory

After a session completes, ingest it into the memory service:

```python
# Complete a session and add to memory
completed_session = await session_service.get_session(
    app_name="my_agent",
    user_id="user_123",
    session_id="session_456",
)

await memory_service.add_session_to_memory(completed_session)
```

### Searching Memory

Query the memory service to retrieve relevant past information:

```python
from google.adk.memory import SearchMemoryResponse

# Search for relevant memories
results: SearchMemoryResponse = await memory_service.search_memory(
    app_name="my_agent",
    user_id="user_123",
    query="What did we discuss about project deadlines?",
)

for memory in results.memories:
    print(f"Session: {memory.session_id}")
    print(f"Content: {memory.content}")
    print(f"Relevance: {memory.score}")
```

---

## Memory Workflow

```
1. User interacts with agent (Session generates events)
         │
         ▼
2. Session completes
         │
         ▼
3. add_session_to_memory(completed_session)
         │
         ▼
4. Memory service extracts and stores key information
         │
         ▼
5. In future sessions, agent queries memory via search_memory()
         │
         ▼
6. Relevant past information incorporated into responses
```

---

## Integration with Runner

Configure memory service when creating the Runner:

```python
from google.adk import Runner
from google.adk.sessions import InMemorySessionService
from google.adk.memory import InMemoryMemoryService

# Create services
session_service = InMemorySessionService()
memory_service = InMemoryMemoryService()

# Configure runner with memory
runner = Runner(
    agent=root_agent,
    app_name="my_agent",
    session_service=session_service,
    memory_service=memory_service,
)
```

---

## Memory Tools for Agents

Agents can access memory through built-in tools like `load_memory`:

```python
from google.adk.agents import Agent
from google.adk.tools import load_memory

root_agent = Agent(
    name="assistant",
    model="gemini-2.5-flash",
    instruction="""You are a helpful assistant with access to conversation history.
    Use the load_memory tool to recall past conversations when relevant.""",
    tools=[load_memory],
)
```

The `load_memory` tool internally calls the memory service's `search_memory()` function.

---

## Use Cases

### 1. User Preferences Across Sessions

```python
# Session 1: User states preference
# User: "I prefer responses in Korean"

# Session 2: Agent recalls preference
# Agent uses load_memory to find: "User prefers Korean responses"
# Agent responds in Korean
```

### 2. Ongoing Project Tracking

```python
# Multiple sessions about a project
# Memory stores: project name, deadlines, decisions made

# New session
# User: "What's the status of my project?"
# Agent queries memory for all project-related information
```

### 3. Personalized Recommendations

```python
# Memory accumulates user interests over time
# - Previous purchases
# - Topics discussed
# - Stated preferences

# Agent uses memory to make personalized suggestions
```

### 4. Multi-Session Task Continuity

```python
# Session 1: Start a complex task
# Session 2: Continue where left off
# Memory provides context from previous sessions
```

---

## Best Practices

### 1. Selective Memory Ingestion

Not all sessions need memory storage:

```python
# Only add meaningful sessions to memory
if session_was_productive(completed_session):
    await memory_service.add_session_to_memory(completed_session)
```

### 2. User-Scoped Memory

Memory is typically scoped per user:

```python
# Each user has their own memory space
results = await memory_service.search_memory(
    app_name="my_agent",
    user_id="user_123",  # User-specific
    query="...",
)
```

### 3. Clear Memory Queries

Write specific queries for better search results:

```python
# ❌ Vague
query = "stuff"

# ✅ Specific
query = "user's preferred programming language"
```

### 4. Graceful Fallback

Handle cases where memory returns no results:

```python
results = await memory_service.search_memory(...)

if results.memories:
    # Use memory context
    context = "\n".join([m.content for m in results.memories])
else:
    # Proceed without memory context
    context = ""
```

---

## Configuration Options

### CLI Configuration

When using `adk api_server`, configure memory service via flags:

```bash
adk api_server backend/agents \
    --memory_service_uri "vertex://project-id/location/agent-engine-id"
```

### Multiple Memory Sources

While only one memory service configures via framework flags, you can manually instantiate additional services:

```python
# Primary memory from framework config
primary_memory = runner.memory_service

# Additional memory source
secondary_memory = VertexAiMemoryBankService(
    project="other-project",
    location="us-central1",
    agent_engine_id="other-engine",
)

# Query both
primary_results = await primary_memory.search_memory(...)
secondary_results = await secondary_memory.search_memory(...)
```

---

## Memory vs State Comparison

| Aspect | State | Memory |
|--------|-------|--------|
| Scope | Single session | Across sessions |
| Access | Direct key-value | Search-based |
| Storage | Session service | Memory service |
| Query | `context.state["key"]` | `search_memory(query)` |
| Use Case | Current conversation context | Long-term knowledge |

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Memory Overview | https://google.github.io/adk-docs/sessions/memory/ |
| Sessions & State | https://google.github.io/adk-docs/sessions/ |
| Vertex AI Memory Bank | https://google.github.io/adk-docs/sessions/memory/#vertexaimemoryservice |
