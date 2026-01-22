# Callbacks

> **Reference**: https://google.github.io/adk-docs/callbacks/

## Overview

Callbacks are functions you define and associate with an agent during creation. They enable observation and control of agent execution at predefined checkpoints, providing a powerful mechanism to hook into an agent's execution process.

### Key Capabilities

| Capability | Description |
|------------|-------------|
| Observability | Log critical execution points for debugging |
| Validation | Inspect and modify data flowing through the agent |
| Guardrails | Enforce policies and prevent disallowed operations |
| State Management | Read or update session state dynamically |
| Integration | Trigger external actions or implement caching |

---

## Types of Callbacks

> **Reference**: https://google.github.io/adk-docs/callbacks/types-of-callbacks/

ADK supports six primary callback points:

| Callback | Trigger Point | Use Cases |
|----------|---------------|-----------|
| `before_agent_callback` | Before agent's `_run_async_impl` begins | Resource setup, state validation, gating |
| `after_agent_callback` | After agent completes successfully | Post-processing, cleanup, final state modifications |
| `before_model_callback` | Before LLM request is sent | Request modification, guardrails, caching |
| `after_model_callback` | After LLM response received | Response validation, content filtering |
| `before_tool_callback` | Before tool execution | Input validation, permission checks |
| `after_tool_callback` | After tool execution completes | Result validation, output logging |

### Control Flow Mechanism

All callbacks follow an "allow/skip/replace" pattern:

- **Returning `None`**: Proceed with standard operations
- **Returning a value**: Override default behavior with replacement data

---

## Agent Callbacks

### before_agent_callback

Executes immediately before the agent's core processing begins.

```python
from google.adk.agents import LlmAgent
from google.adk.agents.callback_context import CallbackContext
from google.genai import types

def before_agent_callback(callback_context: CallbackContext) -> types.Content | None:
    """Called before agent execution starts."""
    # Access context information
    agent_name = callback_context.agent_name
    invocation_id = callback_context.invocation_id

    # Check conditions
    if callback_context.state.get("user:blocked"):
        # Skip agent execution and return early
        return types.Content(
            role="model",
            parts=[types.Part(text="Access denied.")]
        )

    # Return None to proceed normally
    return None

agent = LlmAgent(
    name="my_agent",
    model="gemini-2.5-flash",
    before_agent_callback=before_agent_callback,
)
```

### after_agent_callback

Fires immediately after the agent successfully completes its execution.

```python
def after_agent_callback(callback_context: CallbackContext) -> types.Content | None:
    """Called after agent execution completes."""
    # Perform cleanup
    callback_context.state["session:last_completed"] = callback_context.invocation_id

    # Return None to use agent's original output
    # Return Content to replace the output
    return None

agent = LlmAgent(
    name="my_agent",
    model="gemini-2.5-flash",
    after_agent_callback=after_agent_callback,
)
```

---

## Model Callbacks

### before_model_callback

Executes just before sending requests to the LLM.

```python
from google.adk.agents import LlmAgent
from google.adk.agents.callback_context import CallbackContext
from google.adk.models import LlmRequest, LlmResponse
from google.genai import types

def before_model_callback(
    callback_context: CallbackContext,
    llm_request: LlmRequest
) -> LlmResponse | None:
    """Intercept and modify LLM requests."""
    # Access the request
    contents = llm_request.contents
    system_instruction = llm_request.config.system_instruction

    # Implement guardrails
    last_message = contents[-1].parts[0].text if contents else ""
    if "forbidden_word" in last_message.lower():
        return LlmResponse(
            content=types.Content(
                role="model",
                parts=[types.Part(text="I cannot process that request.")]
            )
        )

    # Modify request (optional)
    llm_request.config.system_instruction += "\nBe concise."

    # Return None to proceed with (modified) request
    return None

agent = LlmAgent(
    name="my_agent",
    model="gemini-2.5-flash",
    before_model_callback=before_model_callback,
)
```

### after_model_callback

Executes immediately after the LLM returns a response.

```python
def after_model_callback(
    callback_context: CallbackContext,
    llm_response: LlmResponse
) -> LlmResponse | None:
    """Process and modify LLM responses."""
    # Access the response
    response_text = llm_response.content.parts[0].text

    # Log for monitoring
    print(f"LLM Response: {response_text[:100]}...")

    # Modify response if needed
    if "sensitive_info" in response_text:
        return LlmResponse(
            content=types.Content(
                role="model",
                parts=[types.Part(text="[Content filtered]")]
            )
        )

    # Return None to use original response
    return None

agent = LlmAgent(
    name="my_agent",
    model="gemini-2.5-flash",
    after_model_callback=after_model_callback,
)
```

---

## Tool Callbacks

### before_tool_callback

Fires just before a tool/function is executed.

```python
from google.adk.agents import LlmAgent
from google.adk.agents.callback_context import CallbackContext
from google.adk.tools import BaseTool

def before_tool_callback(
    callback_context: CallbackContext,
    tool: BaseTool,
    args: dict
) -> dict | None:
    """Intercept tool execution."""
    tool_name = tool.name

    # Log tool invocation
    print(f"Executing tool: {tool_name} with args: {args}")

    # Permission check
    if tool_name == "delete_data" and not callback_context.state.get("user:admin"):
        return {"error": "Permission denied"}

    # Validate/modify arguments
    if "user_id" in args:
        args["user_id"] = args["user_id"].strip()

    # Return None to proceed, or dict to skip tool and use as result
    return None

agent = LlmAgent(
    name="my_agent",
    model="gemini-2.5-flash",
    tools=[my_tool],
    before_tool_callback=before_tool_callback,
)
```

### after_tool_callback

Executes immediately after tool execution completes.

```python
def after_tool_callback(
    callback_context: CallbackContext,
    tool: BaseTool,
    args: dict,
    result: dict
) -> dict | None:
    """Process tool results."""
    tool_name = tool.name

    # Log result
    print(f"Tool {tool_name} returned: {result}")

    # Validate result
    if "error" in result:
        callback_context.state["session:last_error"] = result["error"]

    # Modify result if needed
    if "sensitive_field" in result:
        del result["sensitive_field"]
        return result

    # Return None to use original result
    return None

agent = LlmAgent(
    name="my_agent",
    model="gemini-2.5-flash",
    tools=[my_tool],
    after_tool_callback=after_tool_callback,
)
```

---

## CallbackContext

`CallbackContext` provides access to agent metadata and session state within callbacks.

### Key Attributes

| Attribute | Description |
|-----------|-------------|
| `agent_name` | Name of the current agent |
| `invocation_id` | Unique ID for the current invocation |
| `state` | Mutable dictionary for session state |

### Key Methods

| Method | Description |
|--------|-------------|
| `save_artifact(filename, part)` | Save file or data blob |
| `load_artifact(filename)` | Load saved artifact |
| `list_artifacts()` | List available artifacts |

### Usage Example

```python
def my_callback(callback_context: CallbackContext) -> None:
    # Read state
    user_count = callback_context.state.get("user:request_count", 0)

    # Write state (automatically tracked in Event.actions.state_delta)
    callback_context.state["user:request_count"] = user_count + 1

    # Save artifact
    callback_context.save_artifact(
        "log.txt",
        types.Part(text=f"Request {user_count + 1}")
    )

    return None
```

---

## Design Patterns and Best Practices

> **Reference**: https://google.github.io/adk-docs/callbacks/design-patterns-and-best-practices/

### Design Patterns

| Pattern | Description | Callback |
|---------|-------------|----------|
| Guardrails | Validate content against policies | `before_model_callback`, `before_tool_callback` |
| State Management | Read/modify session data | All callbacks |
| Logging | Capture structured logs | All callbacks |
| Caching | Store/retrieve results | `before_*` and `after_*` pairs |
| Request Modification | Alter LLM requests | `before_model_callback` |
| Response Filtering | Transform LLM responses | `after_model_callback` |
| Conditional Skipping | Bypass operations based on state | `before_*` callbacks |

### Guardrails Example

```python
BLOCKED_TOPICS = ["violence", "illegal", "harmful"]

def guardrail_callback(
    callback_context: CallbackContext,
    llm_request: LlmRequest
) -> LlmResponse | None:
    """Block requests containing prohibited topics."""
    last_message = llm_request.contents[-1].parts[0].text.lower()

    for topic in BLOCKED_TOPICS:
        if topic in last_message:
            return LlmResponse(
                content=types.Content(
                    role="model",
                    parts=[types.Part(text="I cannot discuss that topic.")]
                )
            )
    return None
```

### Caching Example

```python
import hashlib

CACHE = {}

def caching_before_callback(
    callback_context: CallbackContext,
    llm_request: LlmRequest
) -> LlmResponse | None:
    """Check cache before LLM call."""
    cache_key = hashlib.md5(str(llm_request.contents).encode()).hexdigest()
    callback_context.state["temp:cache_key"] = cache_key

    if cache_key in CACHE:
        return CACHE[cache_key]
    return None

def caching_after_callback(
    callback_context: CallbackContext,
    llm_response: LlmResponse
) -> LlmResponse | None:
    """Store result in cache after LLM call."""
    cache_key = callback_context.state.get("temp:cache_key")
    if cache_key:
        CACHE[cache_key] = llm_response
    return None
```

### Best Practices

| Practice | Description |
|----------|-------------|
| Single Purpose | Design each callback for one well-defined purpose |
| Performance | Avoid blocking operations; callbacks execute synchronously |
| Error Handling | Use try-catch blocks and log errors appropriately |
| State Keys | Use prefixed keys (e.g., `user:`, `session:`) to avoid collisions |
| Idempotency | Design for idempotency when callbacks have side effects |
| Testing | Unit test with mock contexts |
| Documentation | Add clear docstrings to all callbacks |

---

## Complete Example

```python
from google.adk.agents import LlmAgent
from google.adk.agents.callback_context import CallbackContext
from google.adk.models import LlmRequest, LlmResponse
from google.adk.tools import FunctionTool, ToolContext
from google.genai import types

# Define a tool
def search_database(query: str, tool_context: ToolContext) -> dict:
    """Search the database for information."""
    return {"results": [f"Result for: {query}"]}

# Agent callbacks
def before_agent(ctx: CallbackContext) -> types.Content | None:
    ctx.state["session:started_at"] = "2024-01-01T00:00:00Z"
    print(f"Agent {ctx.agent_name} starting...")
    return None

def after_agent(ctx: CallbackContext) -> types.Content | None:
    print(f"Agent {ctx.agent_name} completed")
    return None

# Model callbacks
def before_model(ctx: CallbackContext, req: LlmRequest) -> LlmResponse | None:
    print(f"Sending request to LLM...")
    return None

def after_model(ctx: CallbackContext, res: LlmResponse) -> LlmResponse | None:
    print(f"Received response from LLM")
    return None

# Tool callbacks
def before_tool(ctx: CallbackContext, tool, args) -> dict | None:
    print(f"Executing {tool.name} with {args}")
    return None

def after_tool(ctx: CallbackContext, tool, args, result) -> dict | None:
    print(f"Tool {tool.name} returned {result}")
    return None

# Create agent with all callbacks
agent = LlmAgent(
    name="full_callback_agent",
    model="gemini-2.5-flash",
    instruction="You are a helpful assistant.",
    tools=[FunctionTool(func=search_database)],
    before_agent_callback=before_agent,
    after_agent_callback=after_agent,
    before_model_callback=before_model,
    after_model_callback=after_model,
    before_tool_callback=before_tool,
    after_tool_callback=after_tool,
)
```

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Callbacks Overview | https://google.github.io/adk-docs/callbacks/ |
| Types of Callbacks | https://google.github.io/adk-docs/callbacks/types-of-callbacks/ |
| Design Patterns | https://google.github.io/adk-docs/callbacks/design-patterns-and-best-practices/ |
