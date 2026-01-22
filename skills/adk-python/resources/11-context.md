# Context

> **Official Docs**: https://google.github.io/adk-docs/context/

## Overview

Context is a **bundle of information used by agents and tools during task execution**. It provides background information and resources needed throughout the entire cycle (invocation) from a single user request to the final response.

### Core Functions

| Function | Description |
|----------|-------------|
| State Persistence | Remembers conversation details across multiple steps |
| Data Sharing | Passes information discovered in one step to the next |
| Service Access | Uses framework features like Artifact storage, Memory, Authentication |
| Identity Tracking | Logging/debugging via agent_name, invocation_id |

---

## Context Types

ADK provides four context types, each designed for specific use cases:

| Type | Use Case | Read | Write | Special Features |
|------|----------|------|-------|------------------|
| **InvocationContext** | `_run_async_impl` (CustomAgent) | Yes | Yes | Full state access |
| **ReadonlyContext** | `InstructionProvider` | Yes | No | Read-only access |
| **CallbackContext** | Lifecycle callbacks | Yes | Yes | + Artifact interaction |
| **ToolContext** | `FunctionTool` | Yes | Yes | + Auth + Memory search |

### InvocationContext

Used in custom agent implementations. Provides full access to all context features.

```python
from google.adk.agents import BaseAgent
from google.adk.agents.invocation_context import InvocationContext

class MyCustomAgent(BaseAgent):
    async def _run_async_impl(self, ctx: InvocationContext):
        # Full access to session, state, events
        user_id = ctx.session.id
        current_state = ctx.session.state
        yield event
```

### ReadonlyContext

Passed to instruction provider functions. Only read operations allowed.

```python
from google.adk.agents import Agent
from google.adk.agents.readonly_context import ReadonlyContext

def get_instruction(ctx: ReadonlyContext) -> str:
    # Read state values
    language = ctx.state.get("user:language", "en")
    return f"Respond in {language}."

agent = Agent(
    name="localized_agent",
    instruction=get_instruction,  # Function receives ReadonlyContext
)
```

### CallbackContext

Used in lifecycle callbacks (before_agent_callback, after_agent_callback, etc.).

```python
from google.adk.agents.callback_context import CallbackContext

async def before_agent(ctx: CallbackContext):
    # Read and write state
    ctx.state["invocation_started"] = True

    # Artifact operations
    await ctx.save_artifact("log.txt", Part.from_text("Started"))
    artifacts = await ctx.list_artifacts()
```

### ToolContext

Most feature-rich context, available in tool functions.

```python
from google.adk.tools import ToolContext

def search_knowledge(query: str, tool_context: ToolContext) -> dict:
    """Searches the knowledge base.

    Args:
        query: Search query string

    Returns:
        Search results
    """
    # State operations
    count = tool_context.state.get("search_count", 0)
    tool_context.state["search_count"] = count + 1

    # Memory search
    memories = tool_context.search_memory(query)

    # Authentication (if configured)
    # creds = tool_context.get_auth_response(auth_config)

    return {"results": [...], "memory_context": memories}
```

---

## Common Properties and Methods

### All Context Types

```python
# Identifiers
agent_name = context.agent_name       # Current agent name
invocation_id = context.invocation_id # Unique invocation ID

# Read state
value = context.state.get("key", default)
```

### CallbackContext / ToolContext

| Method | Description |
|--------|-------------|
| `state` | Read/write session state (mutable dict) |
| `save_artifact(filename, part)` | Save file/data |
| `load_artifact(filename)` | Load saved file/data |
| `list_artifacts()` | List available artifacts |

### ToolContext Only

| Method | Description |
|--------|-------------|
| `request_credential(auth_config)` | Request user credentials |
| `get_auth_response(auth_config)` | Retrieve provided credentials |
| `search_memory(query)` | Search past interactions or external knowledge |

---

## Context Caching

> **Official Docs**: https://google.github.io/adk-docs/context/caching/
> **Supported**: ADK Python v1.15.0+, Gemini 2.0+

Context caching reuses large instructions or data across repeated requests, providing **faster response times** and **reduced token costs**.

### How It Works

1. ADK caches the system instruction and initial context
2. Subsequent requests reuse cached content instead of re-processing
3. Cache expires based on TTL or usage count

### Configuration

```python
from google.adk.apps.app import App
from google.adk.agents.context_cache_config import ContextCacheConfig

app = App(
    name='my-agent',
    root_agent=root_agent,
    context_cache_config=ContextCacheConfig(
        min_tokens=2048,      # Minimum tokens to trigger caching
        ttl_seconds=600,      # Cache duration (10 minutes)
        cache_intervals=5,    # Max reuse count before refresh
    ),
)
```

### Configuration Parameters

| Parameter | Description | Default | Recommendation |
|-----------|-------------|---------|----------------|
| `min_tokens` | Minimum token count to activate caching | 0 | 1024-4096 |
| `ttl_seconds` | Cache lifetime in seconds | 1800 (30min) | Based on data freshness needs |
| `cache_intervals` | Max reuse count before cache expires | 10 | 5-20 |

### When to Use

| Scenario | Caching Benefit |
|----------|-----------------|
| Long system instructions | High - reduces repeated processing |
| Large static context (docs, rules) | High - significant cost savings |
| Frequently accessed agents | High - faster response times |
| Dynamic, rapidly changing context | Low - cache invalidation overhead |
| Short instructions | Low - overhead may exceed benefit |

### Cost-Benefit Analysis

```python
# Estimate: 10KB instruction = ~2500 tokens
# Without caching: 2500 tokens * $0.0001/token * 100 requests = $25
# With caching: 2500 tokens * $0.0001/token * 1 + cache hits = ~$2.50

# Rule of thumb: Enable caching when instruction > 1000 tokens
# and agent receives > 10 requests within TTL window
```

---

## Context Compaction

> **Official Docs**: https://google.github.io/adk-docs/context/compaction/
> **Supported**: ADK Python v1.16.0+

Context compaction **summarizes previous workflow event history** to reduce context size. Uses a sliding window approach where compression triggers at specified event counts.

### How It Works

1. Events accumulate during conversation
2. At `compaction_interval`, older events are summarized
3. Summary replaces detailed events, keeping `overlap_size` recent events
4. Process repeats as conversation continues

```
Events: [E1, E2, E3, E4, E5, E6]
                    ↓ compaction_interval=3, overlap_size=1
Compacted: [Summary(E1-E5), E5, E6]
```

### Configuration

```python
from google.adk.apps.app import App, EventsCompactionConfig

app = App(
    name='my-agent',
    root_agent=root_agent,
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=3,  # Compact every 3 events
        overlap_size=1,         # Keep 1 event from previous batch
    ),
)
```

### Configuration Parameters

| Parameter | Description | Recommendation |
|-----------|-------------|----------------|
| `compaction_interval` | Event count that triggers compaction | 5-10 for long conversations |
| `overlap_size` | Events to include from previous compaction | 1-2 for context continuity |
| `summarizer` | (Optional) Custom AI model for summarization | Default uses agent's model |

### Custom Summarizer

Use a dedicated model for summarization to optimize costs or quality:

```python
from google.adk.apps.llm_event_summarizer import LlmEventSummarizer
from google.adk.models import Gemini

# Use a smaller, faster model for summarization
summarization_llm = Gemini(model="gemini-2.5-flash-lite")
my_summarizer = LlmEventSummarizer(llm=summarization_llm)

app = App(
    name='my-agent',
    root_agent=root_agent,
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=5,
        overlap_size=2,
        summarizer=my_summarizer,
    ),
)
```

### Custom Summarizer with Prompt

```python
from google.adk.apps.llm_event_summarizer import LlmEventSummarizer
from google.adk.models import Gemini

custom_prompt = """Summarize the following conversation events concisely.
Focus on: user intent, key decisions, action results.
Omit: greetings, filler words, redundant information.

Events:
{events}

Summary:"""

summarizer = LlmEventSummarizer(
    llm=Gemini(model="gemini-2.5-flash"),
    prompt_template=custom_prompt,
)
```

### When to Use

| Scenario | Compaction Benefit |
|----------|-------------------|
| Long multi-turn conversations | High - prevents context overflow |
| Workflow agents with many steps | High - maintains relevant context |
| Short, single-turn interactions | Low - no benefit |
| Context where details matter | Caution - may lose important nuances |

---

## Combining Caching and Compaction

For optimal performance, use both features together:

```python
from google.adk.apps.app import App, EventsCompactionConfig
from google.adk.agents.context_cache_config import ContextCacheConfig

app = App(
    name='optimized-agent',
    root_agent=root_agent,

    # Cache large static instructions
    context_cache_config=ContextCacheConfig(
        min_tokens=2048,
        ttl_seconds=1800,
        cache_intervals=10,
    ),

    # Compact dynamic conversation history
    events_compaction_config=EventsCompactionConfig(
        compaction_interval=5,
        overlap_size=2,
    ),
)
```

### Optimization Strategy

| Component | Optimization |
|-----------|--------------|
| System instruction | Cache (static, rarely changes) |
| Tool definitions | Cache (static) |
| Conversation history | Compact (dynamic, grows over time) |
| User preferences | State (persisted separately) |

---

## Best Practices

### 1. Choose Appropriate Context Type

```python
# For tools: Use ToolContext
def my_tool(query: str, tool_context: ToolContext) -> dict:
    ...

# For dynamic instructions: Use ReadonlyContext
def get_instruction(ctx: ReadonlyContext) -> str:
    ...

# For callbacks: Use CallbackContext
async def before_agent(ctx: CallbackContext):
    ...
```

### 2. Minimize State Mutations in Tools

```python
# Prefer: Return results, let agent decide
def search(query: str, tool_context: ToolContext) -> dict:
    results = perform_search(query)
    return {"results": results, "count": len(results)}

# Avoid: Excessive state mutations
def search_bad(query: str, tool_context: ToolContext) -> dict:
    tool_context.state["last_query"] = query
    tool_context.state["query_count"] = ...
    tool_context.state["last_results"] = ...  # Too much state
```

### 3. Use Artifacts for Large Data

```python
# State: Small, frequently accessed values
tool_context.state["selected_doc_id"] = "doc_123"

# Artifacts: Large, occasionally accessed data
await tool_context.save_artifact(
    "search_results.json",
    Part.from_text(json.dumps(large_results))
)
```

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Context Overview | https://google.github.io/adk-docs/context/ |
| Context Caching | https://google.github.io/adk-docs/context/caching/ |
| Context Compaction | https://google.github.io/adk-docs/context/compaction/ |
