# Model Context Protocol (MCP)

> **Official Docs**: https://google.github.io/adk-docs/mcp/

## Overview

Model Context Protocol (MCP) is an open standard that standardizes how LLMs communicate with external applications, data sources, and tools. It provides a universal connection mechanism for obtaining context, executing actions, and interacting with various systems.

### MCP Architecture

```
┌─────────────┐         ┌─────────────┐
│  MCP Client │◄───────►│  MCP Server │
│  (ADK Agent)│  stdio  │  (Tools)    │
└─────────────┘   or    └─────────────┘
                 HTTP
```

### ADK MCP Capabilities

| Capability | Description |
|------------|-------------|
| **Use MCP Servers** | ADK agents act as MCP clients, consuming tools from MCP servers |
| **Expose ADK Tools** | Build MCP servers that wrap ADK tools for any MCP client |

---

## MCPToolset

The `MCPToolset` class is ADK's primary mechanism for integrating MCP server tools.

### Key Features

- Automatic MCP server connection management
- Tool discovery and adaptation to ADK format
- Tool call proxying

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `connection_params` | `StdioConnectionParams` | Local process communication (stdio) |
| `connection_params` | `SseConnectionParams` | Remote HTTP server (SSE) |
| `connection_params` | `StreamableHTTPConnectionParams` | Remote HTTP server |
| `tool_filter` | `list[str]` | Restrict which tools are exposed |
| `timeout` | `int` | Connection timeout in seconds |

---

## Connection Types

### 1. Stdio Connection (Local MCP Servers)

For local MCP servers running as subprocess (e.g., npm packages).

```python
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset
from mcp import StdioServerParameters
from google.adk.tools.mcp_tool.mcp_toolset import StdioConnectionParams

toolset = MCPToolset(
    connection_params=StdioConnectionParams(
        server_params=StdioServerParameters(
            command="npx",
            args=["-y", "@modelcontextprotocol/server-filesystem", "/path/to/folder"],
            env={"API_KEY": "value"},  # Optional
        ),
        timeout=5,
    ),
    tool_filter=["read_file", "write_file"],  # Optional: filter tools
)
```

### 2. HTTP Connection (Remote MCP Servers)

For remote MCP servers accessible via HTTP.

```python
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset
from google.adk.tools.mcp_tool.mcp_toolset import StreamableHTTPConnectionParams

toolset = MCPToolset(
    connection_params=StreamableHTTPConnectionParams(
        url="https://your-mcp-server.run.app/mcp",
        headers={"Authorization": "Bearer your_token"},
    ),
)
```

---

## Using MCP Tools in Agents

### Basic Usage with adk web

```python
from google.adk.agents import Agent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StdioConnectionParams
from mcp import StdioServerParameters

# Define MCP toolset
filesystem_tools = MCPToolset(
    connection_params=StdioConnectionParams(
        server_params=StdioServerParameters(
            command="npx",
            args=["-y", "@modelcontextprotocol/server-filesystem", "./data"],
        ),
    ),
)

root_agent = Agent(
    name="file_assistant",
    model="gemini-2.5-flash",
    instruction="Help users read and write files.",
    tools=[filesystem_tools],
)
```

### Async Pattern (for custom runners)

```python
from google.adk.agents import Agent
from google.adk.tools.mcp_tool.mcp_toolset import MCPToolset, StdioConnectionParams
from mcp import StdioServerParameters

async def get_agent_async():
    """Create agent with MCP tools asynchronously."""
    toolset = MCPToolset(
        connection_params=StdioConnectionParams(
            server_params=StdioServerParameters(
                command="npx",
                args=["-y", "@modelcontextprotocol/server-filesystem", "./data"],
            ),
        ),
    )

    agent = Agent(
        name="file_assistant",
        model="gemini-2.5-flash",
        instruction="Help users with file operations.",
        tools=[toolset],
    )

    return agent, toolset


async def main():
    agent, toolset = await get_agent_async()
    try:
        # Use agent...
        pass
    finally:
        await toolset.close()  # Important: cleanup
```

---

## Available MCP Servers

### Official MCP Servers

| Server | Package | Description |
|--------|---------|-------------|
| Filesystem | `@modelcontextprotocol/server-filesystem` | Read/write files |
| GitHub | `@modelcontextprotocol/server-github` | GitHub API access |
| Slack | `@modelcontextprotocol/server-slack` | Slack integration |
| Google Drive | `@modelcontextprotocol/server-gdrive` | Google Drive access |
| PostgreSQL | `@modelcontextprotocol/server-postgres` | Database queries |

### MCP Toolbox for Databases

A production-ready MCP server for database access:

| Category | Supported Sources |
|----------|-------------------|
| Google Cloud | BigQuery, AlloyDB, Spanner, Cloud SQL, Firestore, Bigtable |
| SQL | PostgreSQL, MySQL, SQL Server, ClickHouse, SQLite |
| NoSQL | MongoDB, Redis, Cassandra, Couchbase |
| Graph | Neo4j, Dgraph |
| Other | Looker, Trino, HTTP |

### Google Cloud Genmedia MCP

MCP servers for generative media:
- **Imagen**: Image generation
- **Veo**: Video generation
- **Chirp**: Speech synthesis
- **Lyria**: Music generation

---

## Creating MCP Servers with ADK Tools

Expose ADK tools as MCP server for any MCP client.

```python
import json
from mcp.server.lowlevel import Server
from mcp import types as mcp_types
from google.adk.tools.mcp_tool.conversion_utils import adk_to_mcp_tool_type
from google.adk.tools.function_tool import FunctionTool


def get_weather(city: str) -> str:
    """Get weather for a city.

    Args:
        city: City name

    Returns:
        Weather information
    """
    return f"{city}: Sunny, 25C"


# Create MCP server
app = Server("weather-mcp-server")

# Wrap ADK tool
adk_tool = FunctionTool(get_weather)


@app.list_tools()
async def list_tools():
    """List available tools."""
    return [adk_to_mcp_tool_type(adk_tool)]


@app.call_tool()
async def call_tool(name: str, arguments: dict):
    """Execute a tool."""
    if name == adk_tool.name:
        result = await adk_tool.run_async(args=arguments, tool_context=None)
        return [mcp_types.TextContent(type="text", text=json.dumps(result))]
    raise ValueError(f"Unknown tool: {name}")


# Run with: python -m mcp.server.stdio weather_server:app
```

---

## Deployment Patterns

### 1. Self-Contained Stdio

Package MCP server in agent container.

```dockerfile
FROM python:3.12
RUN npm install -g @modelcontextprotocol/server-filesystem
COPY . /app
WORKDIR /app
CMD ["adk", "api_server", "backend/agents"]
```

### 2. Remote HTTP Server

Deploy MCP as separate Cloud Run service.

```python
# Agent connects to remote MCP
toolset = MCPToolset(
    connection_params=StreamableHTTPConnectionParams(
        url="https://mcp-server-xxxxx.run.app/mcp",
        headers={"Authorization": f"Bearer {os.getenv('MCP_TOKEN')}"},
    ),
)
```

### 3. Kubernetes Sidecar

Run MCP server as sidecar container in GKE.

```yaml
# deployment.yaml
spec:
  containers:
    - name: agent
      image: your-agent-image
    - name: mcp-server
      image: your-mcp-server-image
      ports:
        - containerPort: 8080
```

---

## Best Practices

### 1. Use tool_filter for Security

```python
# Expose only needed tools
toolset = MCPToolset(
    connection_params=StdioConnectionParams(...),
    tool_filter=["read_file"],  # Exclude write_file
)
```

### 2. Restrict Filesystem Paths

```python
import os

# Use absolute paths
safe_path = os.path.dirname(os.path.abspath(__file__))

toolset = MCPToolset(
    connection_params=StdioConnectionParams(
        server_params=StdioServerParameters(
            command="npx",
            args=["-y", "@modelcontextprotocol/server-filesystem", safe_path],
        ),
    ),
)
```

### 3. Set Connection Timeouts

```python
toolset = MCPToolset(
    connection_params=StdioConnectionParams(
        server_params=StdioServerParameters(...),
        timeout=10,  # 10 second timeout
    ),
)
```

### 4. Proper Resource Cleanup

```python
async def main():
    agent, toolset = await get_agent_async()
    try:
        # Use agent
        pass
    finally:
        await toolset.close()  # Always cleanup
```

---

## Limitations

| Limitation | Description |
|------------|-------------|
| **Statefulness** | Scaling challenges with stateful MCP servers |
| **Stdio Overhead** | Each connection spawns a subprocess |
| **Deployment** | Async patterns fail in deployment; use sync definitions |
| **Sessions** | Remote HTTP needs careful session management |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Connection timeout | Increase `timeout` parameter |
| Tool not found | Check `tool_filter` configuration |
| npx not found | Ensure Node.js is installed |
| Permission denied | Check filesystem paths and permissions |
| Memory issues | Use remote HTTP instead of many stdio connections |

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| MCP Overview | https://google.github.io/adk-docs/mcp/ |
| MCP Tools | https://google.github.io/adk-docs/tools-custom/mcp-tools/ |
| MCP Toolbox for Databases | https://google.github.io/adk-docs/mcp/mcp-toolbox/ |
| FastMCP | https://google.github.io/adk-docs/mcp/fastmcp/ |
| MCP Specification | https://modelcontextprotocol.io/ |
