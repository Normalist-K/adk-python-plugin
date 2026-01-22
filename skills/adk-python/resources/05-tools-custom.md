# Custom Tools

> **Reference**: https://google.github.io/adk-docs/tools-custom/

## Overview

ADK allows you to create custom tools that extend agent capabilities. This guide covers Function Tools, MCP integration, OpenAPI tools, and authentication.

---

## Function Tools

> **Reference**: https://google.github.io/adk-docs/tools-custom/function-tools/

Python functions become agent tools automatically when passed to the `tools` parameter.

### Basic Structure

```python
from google.adk.agents import Agent

def get_weather(city: str) -> str:
    """Returns the current weather for a city.

    Args:
        city: The name of the city to get weather for

    Returns:
        Weather information for the specified city
    """
    # API call logic
    return f"Weather in {city}: Sunny, 25°C"

agent = Agent(
    name="weather_agent",
    model="gemini-2.5-flash",
    tools=[get_weather],  # Automatically wrapped as FunctionTool
)
```

### Google-Style Docstring Requirement

LLM learns tool usage from docstrings. **Clear, detailed docstrings are mandatory**.

```python
def search_documents(
    query: str,
    max_results: int = 5,
    include_metadata: bool = False
) -> dict:
    """Searches the document database for relevant documents.

    Retrieves documents matching the search query, sorted by relevance.
    Use this tool when you need to find information from the knowledge base.

    Args:
        query: The search question or keywords to find documents
        max_results: Maximum number of results to return (default: 5)
        include_metadata: Whether to include document metadata (default: False)

    Returns:
        A dictionary containing search results:
        - status: Whether the search succeeded
        - results: List of matching documents
        - count: Number of results found
    """
    results = perform_search(query, max_results)
    return {
        "status": "success",
        "results": results,
        "count": len(results),
    }
```

### Type Hints

| Type | Description |
|------|-------------|
| `str`, `int`, `float`, `bool` | Basic types (recommended) |
| `list[str]`, `dict[str, Any]` | Complex types |
| `Optional[str]` | Optional parameters |

```python
# Good - clear type hints
def process(text: str, count: int = 10) -> dict:
    ...

# Bad - will be ignored by ADK
def process(*args, **kwargs):  # Variadic args not supported
    ...
```

### Return Types

**Dictionary returns are recommended:**

```python
# Recommended - explicit structure
def get_data() -> dict:
    return {
        "status": "success",
        "data": result,
        "message": "Processing complete"
    }

# Also valid - auto-wrapped
def get_count() -> int:
    return 42  # Becomes {"result": 42}
```

---

## ToolContext Parameter

> **Reference**: https://google.github.io/adk-docs/tools-custom/function-tools/#toolcontext

`ToolContext` provides access to session state and control signals. It is **automatically injected** when included as a parameter.

```python
from google.adk.tools import ToolContext

def save_preference(
    key: str,
    value: str,
    tool_context: ToolContext  # Auto-injected, not exposed to LLM
) -> dict:
    """Saves a user preference."""
    tool_context.state[key] = value
    return {"status": "saved", "key": key}

def get_preference(
    key: str,
    tool_context: ToolContext
) -> str:
    """Retrieves a saved preference."""
    return tool_context.state.get(key, "Not set")
```

### ToolContext Capabilities

| Attribute/Method | Purpose |
|-----------------|---------|
| `tool_context.state` | Access session state dictionary |
| `tool_context.actions.escalate` | Signal to exit LoopAgent |
| `tool_context.request_confirmation()` | Request user confirmation |
| `tool_context.function_call_id` | Current function call identifier |

### Escalate Pattern (LoopAgent Exit)

```python
def exit_loop(
    reason: str,
    tool_context: ToolContext
) -> dict:
    """Exits the loop. Call when quality criteria are met."""
    tool_context.actions.escalate = True  # Exit LoopAgent
    return {"status": "completed", "reason": reason}
```

---

## Tool Performance (Async Tools)

Async tools enable concurrent execution and better performance for I/O-bound operations.

### Async Function Tool

```python
import aiohttp

async def fetch_data(url: str) -> dict:
    """Fetches data from a URL asynchronously."""
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()
```

### Parallel Execution

```python
import asyncio

async def search_multiple(queries: list[str]) -> dict:
    """Searches multiple queries in parallel.

    This function is optimized for concurrent execution.
    """
    tasks = [search_single(q) for q in queries]
    results = await asyncio.gather(*tasks)
    return {"results": results}
```

### CPU-Intensive Tasks

```python
from concurrent.futures import ThreadPoolExecutor
import asyncio

async def cpu_intensive_task(data: str) -> dict:
    """Runs CPU-intensive work in a thread pool."""
    loop = asyncio.get_event_loop()
    with ThreadPoolExecutor() as pool:
        result = await loop.run_in_executor(pool, heavy_computation, data)
    return {"result": result}
```

### Performance Best Practices

| Practice | Benefit |
|----------|---------|
| Use `async` for I/O operations | Non-blocking execution |
| Use `asyncio.gather()` for parallel calls | Concurrent requests |
| Use `ThreadPoolExecutor` for CPU work | Avoid blocking event loop |
| Cache expensive operations | Reduce redundant calls |

---

## Action Confirmations (Human-in-the-Loop)

> **Reference**: https://google.github.io/adk-docs/tools-custom/confirmation/

Request user confirmation before executing sensitive operations.

### Boolean Confirmation

```python
from google.adk.tools import FunctionTool

def delete_file(path: str) -> dict:
    """Deletes a file from the filesystem."""
    import os
    os.remove(path)
    return {"status": "deleted", "path": path}

# Wrap with confirmation requirement
delete_tool = FunctionTool(
    func=delete_file,
    require_confirmation=True,  # User must approve
)

agent = Agent(
    name="file_manager",
    model="gemini-2.5-flash",
    tools=[delete_tool],
)
```

### Advanced Confirmation with Context

```python
from google.adk.tools import ToolContext

def transfer_money(
    amount: float,
    to_account: str,
    tool_context: ToolContext
) -> dict:
    """Transfers money to an account. Requires user confirmation."""
    # Request confirmation with details
    confirmation = tool_context.request_confirmation(
        hint=f"Transfer ${amount} to {to_account}. Proceed?",
        payload={"amount": amount, "to_account": to_account}
    )

    if confirmation.approved:
        execute_transfer(amount, to_account)
        return {"status": "success", "amount": amount}
    else:
        return {"status": "cancelled", "reason": "User declined"}
```

### Confirmation Flow

```
1. Tool called by agent
2. ADK detects confirmation requirement
3. User presented with confirmation prompt
4. User approves or declines
5. Tool executes (if approved) or returns cancellation
```

---

## MCP Tools (Model Context Protocol)

> **Reference**: https://google.github.io/adk-docs/tools-custom/mcp-tools/

Integrate tools from MCP servers for extended capabilities.

### Using MCP Toolset

```python
from google.adk.tools.mcp_tool import MCPToolset, StdioServerParameters

# Connect to an MCP server
mcp_tools = MCPToolset(
    connection_params=StdioServerParameters(
        command="npx",
        args=["-y", "@anthropic-ai/mcp-server-filesystem", "/path/to/dir"]
    )
)

agent = Agent(
    name="mcp_agent",
    model="gemini-2.5-flash",
    tools=[mcp_tools],  # All MCP tools available
)
```

### SSE Connection (HTTP)

```python
from google.adk.tools.mcp_tool import MCPToolset, SseServerParams

mcp_tools = MCPToolset(
    connection_params=SseServerParams(
        url="http://localhost:8080/sse"
    )
)
```

### Available MCP Servers

| Server | Purpose |
|--------|---------|
| `@anthropic-ai/mcp-server-filesystem` | File operations |
| `@anthropic-ai/mcp-server-git` | Git operations |
| `@anthropic-ai/mcp-server-github` | GitHub API |
| `@anthropic-ai/mcp-server-slack` | Slack integration |
| `@anthropic-ai/mcp-server-sqlite` | SQLite database |

### Custom MCP Tool Selection

```python
# Get specific tools from MCP server
mcp_toolset = MCPToolset(
    connection_params=StdioServerParameters(
        command="npx",
        args=["-y", "@some/mcp-server"]
    )
)

# Filter to specific tools
tools, exit_stack = await mcp_toolset.get_tools()
selected_tools = [t for t in tools if t.name in ["read_file", "write_file"]]
```

---

## OpenAPI Tools

> **Reference**: https://google.github.io/adk-docs/tools-custom/openapi-tools/

Generate tools automatically from OpenAPI specifications.

### Basic Usage

```python
from google.adk.tools import OpenAPIToolset

# From OpenAPI spec string
toolset = OpenAPIToolset(
    spec_str=open("api_spec.yaml").read(),
)

agent = Agent(
    name="api_agent",
    model="gemini-2.5-flash",
    tools=[toolset],
)
```

### From URL

```python
toolset = OpenAPIToolset(
    spec_str=requests.get("https://api.example.com/openapi.json").text,
)
```

### Filtering Operations

```python
# Only include specific operations
toolset = OpenAPIToolset(
    spec_str=spec,
    only_operations=["getUser", "createUser", "listUsers"],
)

# Exclude specific operations
toolset = OpenAPIToolset(
    spec_str=spec,
    exclude_operations=["deleteUser", "adminOps"],
)
```

### OpenAPI Tool Features

| Feature | Description |
|---------|-------------|
| Auto-generated docstrings | From OpenAPI descriptions |
| Type inference | From schema definitions |
| Path/Query parameters | Automatic handling |
| Request body | JSON serialization |

---

## Authentication

> **Reference**: https://google.github.io/adk-docs/tools-custom/authentication/

Manage authentication for protected resources and APIs.

### Supported Authentication Types

| Type | Description | Use Case |
|------|-------------|----------|
| `API_KEY` | Static API key | Simple API access |
| `HTTP` | Basic/Bearer token | REST APIs |
| `OAUTH2` | OAuth 2.0 flows | User-authorized access |
| `OPEN_ID_CONNECT` | OpenID Connect | Identity providers |
| `SERVICE_ACCOUNT` | GCP service account | Google Cloud services |

### API Key Authentication

```python
from google.adk.auth import AuthCredential, AuthCredentialTypes, ApiKeyConfig
from google.adk.tools import OpenAPIToolset

auth_credential = AuthCredential(
    auth_type=AuthCredentialTypes.API_KEY,
    api_key=ApiKeyConfig(
        api_key="your_api_key_here",
        header_name="X-API-Key",  # or "Authorization"
    )
)

toolset = OpenAPIToolset(
    spec_str=openapi_spec,
    auth_credential=auth_credential,
)
```

### OAuth2 Configuration

```python
from google.adk.auth import (
    OAuth2, OAuthFlows, AuthCredential,
    AuthCredentialTypes, OAuth2Auth
)

# Define OAuth2 scheme
auth_scheme = OAuth2(
    flows=OAuthFlows(
        authorization_code={
            "authorizationUrl": "https://provider.com/oauth/authorize",
            "tokenUrl": "https://provider.com/oauth/token",
            "scopes": {
                "read": "Read access",
                "write": "Write access"
            }
        }
    )
)

# Credentials
auth_credential = AuthCredential(
    auth_type=AuthCredentialTypes.OAUTH2,
    oauth2=OAuth2Auth(
        client_id="your_client_id",
        client_secret="your_client_secret"
    )
)

# Apply to toolset
toolset = OpenAPIToolset(
    spec_str=openapi_spec,
    auth_scheme=auth_scheme,
    auth_credential=auth_credential,
)
```

### OAuth Flow Handling

```
1. Agent calls tool requiring authentication
2. ADK detects missing/expired token
3. `adk_request_credential` function called
4. User redirected to authorization URL
5. User authorizes and callback received
6. ADK exchanges code for tokens
7. Tool automatically re-executed with valid token
```

### Bearer Token (HTTP Auth)

```python
from google.adk.auth import AuthCredential, AuthCredentialTypes, HttpAuth

auth_credential = AuthCredential(
    auth_type=AuthCredentialTypes.HTTP,
    http=HttpAuth(
        scheme="Bearer",
        credentials="your_access_token"
    )
)
```

### Service Account (GCP)

```python
from google.adk.auth import AuthCredential, AuthCredentialTypes, ServiceAccountConfig

auth_credential = AuthCredential(
    auth_type=AuthCredentialTypes.SERVICE_ACCOUNT,
    service_account=ServiceAccountConfig(
        service_account_email="sa@project.iam.gserviceaccount.com",
        scopes=["https://www.googleapis.com/auth/cloud-platform"]
    )
)
```

---

## Best Practices Summary

### Function Tools

| Practice | Recommendation |
|----------|----------------|
| Docstrings | Always use Google-style with Args/Returns |
| Type hints | Use explicit types, avoid `*args/**kwargs` |
| Return type | Prefer `dict` with status fields |
| Single responsibility | One tool = one action |
| Error handling | Return error info, don't raise exceptions |

### Performance

| Practice | Recommendation |
|----------|----------------|
| I/O operations | Use `async` functions |
| Multiple calls | Use `asyncio.gather()` |
| CPU-intensive | Use `ThreadPoolExecutor` |

### Security

| Practice | Recommendation |
|----------|----------------|
| Sensitive actions | Use `require_confirmation=True` |
| Credentials | Use environment variables |
| OAuth tokens | Let ADK handle refresh |

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Custom Tools Overview | https://google.github.io/adk-docs/tools-custom/ |
| Function Tools | https://google.github.io/adk-docs/tools-custom/function-tools/ |
| MCP Tools | https://google.github.io/adk-docs/tools-custom/mcp-tools/ |
| OpenAPI Tools | https://google.github.io/adk-docs/tools-custom/openapi-tools/ |
| Authentication | https://google.github.io/adk-docs/tools-custom/authentication/ |
