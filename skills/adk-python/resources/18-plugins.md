# Plugins

> **Official Docs**:
> - [Plugins Overview](https://google.github.io/adk-docs/plugins/)
> - [Reflect and Retry Plugin](https://google.github.io/adk-docs/plugins/reflect-and-retry/)

## Overview

Plugins are custom code modules executed at various stages of an agent workflow lifecycle using callback hooks. They provide functionality applicable across entire agent workflows rather than individual agents.

| Use Case | Description |
|----------|-------------|
| Logging/Tracing | Track execution across all agents and tools |
| Policy Enforcement | Validate inputs/outputs globally |
| Monitoring/Metrics | Collect performance data |
| Response Caching | Cache LLM responses for efficiency |
| Request/Response Modification | Transform data at runtime |

### Plugin vs Callback

| Feature | Plugin | Agent Callback |
|---------|--------|----------------|
| Scope | Global (all agents, tools, LLM calls) | Single agent/tool |
| Registration | Once on Runner | Per agent/tool |
| Precedence | Higher (can short-circuit) | Lower |

---

## Plugin Architecture

### Lifecycle Hooks

Plugins support callbacks at these lifecycle stages:

| Stage | Callback | Purpose |
|-------|----------|---------|
| User Input | `on_user_message_callback` | Inspect/modify raw user input |
| Runner Start | `before_run_callback` | Global setup before agent logic |
| Agent | `before_agent_callback`, `after_agent_callback` | Intercept agent operations |
| Model | `before_model_callback`, `after_model_callback`, `on_model_error_callback` | Handle LLM execution and errors |
| Tool | `before_tool_callback`, `after_tool_callback`, `on_tool_error_callback` | Intercept tool operations and failures |
| Events | `on_event_callback` | Modify events before streaming to client |
| Runner End | `after_run_callback` | Cleanup and final reporting |

### Flow Control Modes

Plugins support three operational approaches:

| Mode | Return Value | Effect |
|------|--------------|--------|
| Observe | `None` | Log/collect metrics without interrupting |
| Intervene | Non-null value | Short-circuit and replace the action |
| Amend | Modify context | Change context without interrupting |

---

## Built-in Plugins

ADK provides several ready-to-use plugins:

| Plugin | Description |
|--------|-------------|
| **Reflect and Retry** | Tracks tool failures and intelligently retries |
| **BigQuery Analytics** | Enables agent logging and analysis |
| **Context Filter** | Reduces generative AI context size |
| **Global Instruction** | Provides global instructions at App level |
| **Save Files as Artifacts** | Saves user message files as Artifacts |
| **Logging** | Logs information at each callback point |

---

## Reflect and Retry Plugin

The Reflect and Retry plugin helps agents recover from tool failures by intercepting errors, providing structured guidance to the model for reflection, and automatically retrying operations.

### Key Features

- **Concurrency safe**: Uses locking for parallel tool executions
- **Configurable scope**: Tracks failures per-invocation (default) or globally
- **Granular tracking**: Maintains failure counts per-tool
- **Custom error extraction**: Detects errors in normal tool responses

### Basic Usage

```python
from google.adk.plugins import ReflectAndRetryToolPlugin
from google.adk.app import App

app = App(
    name="my_app",
    root_agent=root_agent,
    plugins=[ReflectAndRetryToolPlugin(max_retries=3)]
)
```

### Configuration Options

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `max_retries` | int | 3 | Additional retry attempts for error responses |
| `throw_exception_if_retry_exceeded` | bool | True | Raise error if retries exhausted |
| `tracking_scope` | Enum | INVOCATION | INVOCATION (per user) or GLOBAL (all users) |

### Advanced: Custom Error Extraction

Override `extract_error_from_result()` to detect errors based on custom response patterns:

```python
from google.adk.plugins import ReflectAndRetryToolPlugin

class MyReflectAndRetry(ReflectAndRetryToolPlugin):
    def extract_error_from_result(self, result: dict) -> str | None:
        # Check for custom error patterns in tool response
        if result.get("status") == "error":
            return result.get("message", "Unknown error")
        if result.get("data") is None:
            return "No data returned"
        return None  # No error detected
```

### Use Cases

| Scenario | Benefit |
|----------|---------|
| API rate limits | Automatic retry with backoff |
| Transient network errors | Graceful recovery |
| Invalid tool arguments | Model reflects and corrects |
| Hallucinated function names | Model learns from error feedback |

---

## Creating Custom Plugins

### Basic Structure

```python
from google.adk.plugins.base_plugin import BasePlugin
from google.adk.agents import BaseAgent
from google.adk.events import Event
from google.adk.tools import BaseTool
from google.adk.callbacks import CallbackContext
from typing import Optional

class MyCustomPlugin(BasePlugin):
    def __init__(self) -> None:
        super().__init__(name="my_custom_plugin")
        # Initialize plugin state
        self.call_count = 0

    async def before_agent_callback(
        self,
        *,
        agent: BaseAgent,
        callback_context: CallbackContext
    ) -> Optional[Event]:
        """Called before each agent execution."""
        self.call_count += 1
        print(f"Agent '{agent.name}' starting (total calls: {self.call_count})")
        return None  # Observe mode - don't interrupt

    async def after_agent_callback(
        self,
        *,
        agent: BaseAgent,
        callback_context: CallbackContext,
        event: Event
    ) -> Optional[Event]:
        """Called after each agent execution."""
        print(f"Agent '{agent.name}' completed")
        return None
```

### Registering Plugins

Plugins integrate via the `Runner` class:

```python
from google.adk.runners import InMemoryRunner

runner = InMemoryRunner(
    agent=root_agent,
    app_name="my_app",
    plugins=[MyCustomPlugin()],
)
```

Or via the `App` class:

```python
from google.adk.app import App

app = App(
    name="my_app",
    root_agent=root_agent,
    plugins=[MyCustomPlugin()],
)
```

### Complete Example: Request Logger Plugin

```python
from google.adk.plugins.base_plugin import BasePlugin
from google.adk.agents import BaseAgent
from google.adk.tools import BaseTool
from google.adk.callbacks import CallbackContext
from google.adk.models import LlmRequest, LlmResponse
from typing import Optional
import json
import datetime

class RequestLoggerPlugin(BasePlugin):
    """Logs all LLM requests and tool calls to a file."""

    def __init__(self, log_file: str = "agent_logs.jsonl") -> None:
        super().__init__(name="request_logger")
        self.log_file = log_file

    def _log(self, event_type: str, data: dict) -> None:
        entry = {
            "timestamp": datetime.datetime.now().isoformat(),
            "event": event_type,
            **data
        }
        with open(self.log_file, "a") as f:
            f.write(json.dumps(entry) + "\n")

    async def before_model_callback(
        self,
        *,
        callback_context: CallbackContext,
        llm_request: LlmRequest
    ) -> Optional[LlmResponse]:
        self._log("model_request", {
            "model": str(llm_request.model),
            "messages_count": len(llm_request.contents or []),
        })
        return None

    async def after_tool_callback(
        self,
        *,
        tool: BaseTool,
        callback_context: CallbackContext,
        tool_response: dict
    ) -> Optional[dict]:
        self._log("tool_call", {
            "tool": tool.name,
            "response_keys": list(tool_response.keys()) if tool_response else [],
        })
        return None
```

### Example: Response Caching Plugin

```python
from google.adk.plugins.base_plugin import BasePlugin
from google.adk.callbacks import CallbackContext
from google.adk.models import LlmRequest, LlmResponse
from typing import Optional
import hashlib

class CachingPlugin(BasePlugin):
    """Caches LLM responses to avoid duplicate calls."""

    def __init__(self) -> None:
        super().__init__(name="caching_plugin")
        self.cache: dict[str, LlmResponse] = {}

    def _get_cache_key(self, request: LlmRequest) -> str:
        # Create hash from request contents
        content_str = str(request.contents)
        return hashlib.md5(content_str.encode()).hexdigest()

    async def before_model_callback(
        self,
        *,
        callback_context: CallbackContext,
        llm_request: LlmRequest
    ) -> Optional[LlmResponse]:
        cache_key = self._get_cache_key(llm_request)
        if cache_key in self.cache:
            print(f"Cache hit for request {cache_key[:8]}...")
            return self.cache[cache_key]  # Intervene - return cached response
        return None

    async def after_model_callback(
        self,
        *,
        callback_context: CallbackContext,
        llm_response: LlmResponse
    ) -> Optional[LlmResponse]:
        # Store in cache (would need access to request for key)
        return None
```

---

## Limitations

**Important**: Plugins are not supported by the ADK web interface.

```bash
# This will NOT work with plugins
adk web backend/agents  # Plugins ignored!

# Use programmatic execution instead
python run_agent.py
```

For plugin-enabled workflows, use programmatic execution:

```python
import asyncio
from google.adk.runners import InMemoryRunner
from google.adk.sessions import Session

async def main():
    runner = InMemoryRunner(
        agent=root_agent,
        app_name="my_app",
        plugins=[MyCustomPlugin()],
    )

    session = Session(app_name="my_app", user_id="user1")

    async for event in runner.run_async(
        session=session,
        user_message="Hello!"
    ):
        print(event)

asyncio.run(main())
```

---

## Best Practices

### 1. Use Observe Mode by Default

```python
# Good: Return None to observe without interrupting
async def before_agent_callback(self, *, agent, callback_context):
    self.log(f"Agent {agent.name} starting")
    return None  # Don't interrupt

# Use Intervene only when necessary
async def before_model_callback(self, *, callback_context, llm_request):
    if self.should_block(llm_request):
        return LlmResponse(...)  # Short-circuit with custom response
    return None
```

### 2. Handle Errors Gracefully

```python
async def on_tool_error_callback(
    self, *, tool, callback_context, error
) -> Optional[dict]:
    self.log_error(tool.name, error)
    # Optionally return fallback response
    if self.has_fallback(tool.name):
        return self.get_fallback(tool.name)
    return None  # Let error propagate
```

### 3. Keep Plugins Stateless When Possible

```python
# Good: Stateless plugin
class LoggingPlugin(BasePlugin):
    async def before_agent_callback(self, *, agent, callback_context):
        # Use external logging service
        await external_logger.log(agent.name)
        return None

# Caution: Stateful plugins need thread safety
class CounterPlugin(BasePlugin):
    def __init__(self):
        super().__init__(name="counter")
        self._lock = asyncio.Lock()
        self._count = 0

    async def before_agent_callback(self, *, agent, callback_context):
        async with self._lock:
            self._count += 1
        return None
```

### 4. Combine Multiple Plugins

```python
runner = InMemoryRunner(
    agent=root_agent,
    app_name="my_app",
    plugins=[
        RequestLoggerPlugin(),
        ReflectAndRetryToolPlugin(max_retries=3),
        CachingPlugin(),
    ],
)
```

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Plugins Overview | https://google.github.io/adk-docs/plugins/ |
| Reflect and Retry | https://google.github.io/adk-docs/plugins/reflect-and-retry/ |
| Callbacks Reference | https://google.github.io/adk-docs/callbacks/ |
