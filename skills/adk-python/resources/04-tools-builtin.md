# Built-in Tools

> **Reference**: https://google.github.io/adk-docs/tools/

## Overview

ADK provides pre-built tools that extend agent capabilities with external functionality.

### Tool Categories

| Category | Description | Examples |
|----------|-------------|----------|
| Gemini API | Google AI capabilities | Google Search, Code Execution, Computer Use |
| Google Cloud | GCP service integrations | BigQuery, Vertex AI Search, Spanner |
| Third-Party | External service integrations | GitHub, Notion, Stripe, PayPal |

### Basic Usage Pattern

```python
from google.adk.agents import Agent
from google.adk.tools import google_search  # 1. Import

agent = Agent(
    name="search_agent",
    model="gemini-2.0-flash",
    tools=[google_search],  # 2. Register
)
```

---

## Gemini API Tools

> **Reference**: https://google.github.io/adk-docs/tools/gemini-api/

### Google Search

> **Reference**: https://google.github.io/adk-docs/tools/gemini-api/google-search/

Performs web searches using Google Search with grounding.

```python
from google.adk.agents import Agent
from google.adk.tools import google_search

search_agent = Agent(
    name="search_agent",
    model="gemini-2.0-flash",  # Gemini 2+ required
    tools=[google_search],
    instruction="Use web search to answer questions with current information.",
)
```

**Requirements:**
- Gemini 2.0+ model required
- `renderedContent` HTML must be displayed in UI (policy requirement)
- Single-tool constraint (see Limitations)

### Code Execution

> **Reference**: https://google.github.io/adk-docs/tools/gemini-api/code-execution/

Executes Python code for calculations and data manipulation.

```python
from google.adk.agents import LlmAgent
from google.adk.code_executors import BuiltInCodeExecutor

code_agent = LlmAgent(
    name="calculator",
    model="gemini-2.0-flash",  # Gemini 2+ required
    code_executor=BuiltInCodeExecutor(),
    instruction="Write Python code to solve calculation problems.",
)
```

**Features:**
- Dynamic code generation and execution
- Supports calculations, data manipulation
- Python language only

**Requirements:**
- Gemini 2.0+ model required
- Single-tool constraint (see Limitations)

### Computer Use

> **Reference**: https://google.github.io/adk-docs/tools/gemini-api/computer-use/

Operates computer interfaces (browsers) via Playwright.

**Installation:**

```bash
pip install termcolor playwright browserbase rich
playwright install-deps chromium
playwright install chromium
```

**Usage:**

```python
from google.adk import Agent
from google.adk.tools.computer_use.computer_use_toolset import ComputerUseToolset

# Implement PlaywrightComputer class (see docs)
root_agent = Agent(
    model="gemini-2.5-computer-use-preview-10-2025",
    tools=[ComputerUseToolset(computer=PlaywrightComputer(screen_size=(1280, 936)))]
)
```

**Requirements:**
- Preview release (ADK Python v1.17.0+)
- Specialized model: `gemini-2.5-computer-use-preview-10-2025`
- Playwright browser automation

---

## Google Cloud Tools

> **Reference**: https://google.github.io/adk-docs/tools/google-cloud/

### BigQuery

> **Reference**: https://google.github.io/adk-docs/tools/google-cloud/bigquery/

Query and analyze data in BigQuery datasets.

| Tool | Description |
|------|-------------|
| `list_dataset_ids` | List dataset identifiers |
| `get_dataset_info` | Get dataset metadata |
| `list_table_ids` | List tables in dataset |
| `get_table_info` | Get table schema |
| `execute_sql` | Run SQL queries |
| `forecast` | Time series predictions |
| `ask_data_insights` | Natural language data queries |

```python
import google.auth
from google.adk.agents import Agent
from google.adk.tools.bigquery import BigQueryToolset
from google.adk.tools.bigquery.config import (
    BigQueryCredentialsConfig,
    BigQueryToolConfig,
    WriteMode
)

# Setup credentials (ADC)
credentials, _ = google.auth.default()
credentials_config = BigQueryCredentialsConfig(credentials=credentials)

# Configure write permissions
tool_config = BigQueryToolConfig(write_mode=WriteMode.BLOCKED)

# Create toolset
bigquery_toolset = BigQueryToolset(
    credentials_config=credentials_config,
    bigquery_tool_config=tool_config
)

agent = Agent(
    model="gemini-2.0-flash",
    tools=[bigquery_toolset]
)
```

**Version**: ADK Python v1.1.0+

### Vertex AI Search

> **Reference**: https://google.github.io/adk-docs/tools/google-cloud/vertex-ai-search/

Search across private data stores configured in Vertex AI Search.

```python
from google.adk.agents import LlmAgent
from google.adk.tools import VertexAiSearchTool

# Datastore path format
datastore = "projects/<PROJECT>/locations/<REGION>/collections/default_collection/dataStores/<DATASTORE_ID>"

search_tool = VertexAiSearchTool(datastore=datastore)

agent = LlmAgent(
    model="gemini-2.0-flash",
    tools=[search_tool],  # Single-tool constraint applies
    instruction="Search internal documents to answer questions."
)
```

**Requirements:**
- Vertex AI Search datastore configured
- Single-tool constraint (see Limitations)
- ADK Python v0.1.0+

### Vertex AI RAG Engine

> **Reference**: https://google.github.io/adk-docs/tools/google-cloud/vertex-ai-rag-engine/

Private data retrieval using Vertex AI RAG Engine.

```python
from google.adk.agents import LlmAgent
from google.adk.tools import vertex_ai_rag_retrieval
import os

rag_tool = vertex_ai_rag_retrieval(
    name="retrieve_rag_documentation",
    description="Retrieve relevant documentation from the RAG corpus",
    rag_resources=[os.environ["RAG_CORPUS"]],  # projects/123/locations/us-central1/ragCorpora/456
    similarity_top_k=10,
    vector_distance_threshold=0.6
)

agent = LlmAgent(
    model="gemini-2.0-flash",
    tools=[rag_tool]  # Single-tool constraint applies
)
```

**Requirements:**
- RAG corpus setup required (see Vertex AI RAG Engine quickstart)
- Single-tool constraint (see Limitations)
- ADK Python v0.1.0+, Java v0.2.0+

### Spanner

> **Reference**: https://google.github.io/adk-docs/tools/google-cloud/spanner/

Interact with Spanner databases.

| Tool | Description |
|------|-------------|
| `list_table_names` | List table names |
| `list_table_indexes` | List table indexes |
| `list_table_index_columns` | List index columns |
| `list_named_schemas` | List schema definitions |
| `get_table_schema` | Get table structure |
| `execute_sql` | Run SQL queries |
| `similarity_search` | Text similarity search |

```python
import google.auth
from google.adk.tools.spanner import (
    SpannerToolset,
    SpannerCredentialsConfig,
    SpannerToolSettings,
    Capabilities
)

credentials, _ = google.auth.default()
credentials_config = SpannerCredentialsConfig(credentials=credentials)
tool_settings = SpannerToolSettings(capabilities=Capabilities.DATA_READ)

spanner_toolset = SpannerToolset(
    credentials_config=credentials_config,
    tool_settings=tool_settings
)
```

**Version**: ADK Python v1.11.0+

### Additional Google Cloud Tools

| Tool | Description | Version |
|------|-------------|---------|
| **Bigtable** | NoSQL database operations | - |
| **Pub/Sub** | Publish/pull messages | - |
| **Cloud SQL** | SQL database operations | - |
| **Apigee API Hub** | Convert APIs to tools | - |
| **Application Integration** | Enterprise app connections | - |
| **GKE Code Executor** | Secure code execution in K8s | - |
| **MCP Toolbox for Databases** | 30+ data source connections | - |

---

## Third-Party Tools

> **Reference**: https://google.github.io/adk-docs/tools/third-party/

Third-party integrations use MCP (Model Context Protocol) servers.

### GitHub

> **Reference**: https://google.github.io/adk-docs/tools/third-party/github/

Repository management, issue/PR automation, code analysis.

```python
from google.adk.agents import Agent
from google.adk.tools.mcp_tool import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import StreamableHTTPServerParams
import os

GITHUB_TOKEN = os.environ["GITHUB_TOKEN"]

root_agent = Agent(
    model="gemini-2.5-pro",
    name="github_agent",
    instruction="Help users with GitHub operations",
    tools=[McpToolset(
        connection_params=StreamableHTTPServerParams(
            url="https://api.githubcopilot.com/mcp/",
            headers={
                "Authorization": f"Bearer {GITHUB_TOKEN}",
                "X-MCP-Toolsets": "all",
                "X-MCP-Readonly": "true"  # Optional: read-only mode
            }
        )
    )]
)
```

**Key Capabilities**: Repository browsing, issue/PR management, code security, Dependabot alerts, 19+ toolset categories

### GitLab

Semantic code search, pipeline inspection, merge request management.

```python
# Similar MCP setup with GitLab MCP server
```

### Notion

> **Reference**: https://google.github.io/adk-docs/tools/third-party/notion/

Workspace search, page/database management.

```python
from google.adk.agents import Agent
from google.adk.tools.mcp_tool import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import StdioConnectionParams
from mcp import StdioServerParameters
import os

NOTION_TOKEN = os.environ["NOTION_TOKEN"]

root_agent = Agent(
    model="gemini-2.5-pro",
    name="notion_agent",
    instruction="Help users manage Notion workspaces",
    tools=[
        McpToolset(
            connection_params=StdioConnectionParams(
                server_params=StdioServerParameters(
                    command="npx",
                    args=["-y", "@notionhq/notion-mcp-server"],
                    env={"NOTION_TOKEN": NOTION_TOKEN}
                ),
                timeout=30,
            ),
        )
    ],
)
```

**Available Tools**: `notion-search`, `notion-fetch`, `notion-create-pages`, `notion-update-page`, `notion-move-pages`, `notion-duplicate-page`, `notion-create-database`, `notion-create-comment`, `notion-get-users`

### Linear

Issue management, project tracking, development workflow automation.

### Stripe

> **Reference**: https://google.github.io/adk-docs/tools/third-party/stripe/

Payment processing, invoicing, subscription management.

```python
from google.adk.agents import Agent
from google.adk.tools.mcp_tool import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import StreamableHTTPServerParams
import os

STRIPE_KEY = os.environ["STRIPE_SECRET_KEY"]

# Remote MCP Server
agent = Agent(
    model="gemini-2.5-pro",
    name="stripe_agent",
    tools=[McpToolset(
        connection_params=StreamableHTTPServerParams(
            url="https://mcp.stripe.com",
            headers={"Authorization": f"Bearer {STRIPE_KEY}"}
        )
    )]
)
```

**Available Tools** (23 total): Account info, balance retrieval, customer management, invoice operations, payment links/intents, subscription management

**Security**: Enable human confirmation for tool actions to mitigate prompt injection.

### PayPal

Payment management, invoice sending, subscription handling.

### Additional Third-Party Tools

| Tool | Description |
|------|-------------|
| **Asana** | Project and task management |
| **Atlassian** | Jira issues, Confluence pages |
| **Hugging Face** | Models, datasets, AI tools |
| **Qdrant** | Semantic vector search |
| **ElevenLabs** | Speech generation, voice cloning |
| **Cartesia** | Audio content generation |
| **n8n** | Workflow automation |
| **Postman** | API collection management |

---

## Tool Limitations

> **Reference**: https://google.github.io/adk-docs/tools/limitations/

### Single-Tool Constraint

These tools cannot coexist with other tools in the same agent:

| Tool | Category |
|------|----------|
| Code Execution | Gemini API |
| Google Search | Gemini API |
| Vertex AI Search | Google Cloud |
| Vertex AI RAG Engine | Google Cloud |

### Workaround 1: AgentTool

Wrap restricted tools in separate agents using `AgentTool`.

```python
from google.adk.agents import Agent
from google.adk.tools import AgentTool, google_search
from google.adk.code_executors import BuiltInCodeExecutor

# Separate agents for restricted tools
search_agent = Agent(
    name="searcher",
    model="gemini-2.0-flash",
    tools=[google_search],
)

code_agent = Agent(
    name="coder",
    model="gemini-2.0-flash",
    code_executor=BuiltInCodeExecutor(),
)

# Wrap as tools
search_tool = AgentTool(search_agent)
code_tool = AgentTool(code_agent)

# Use together in main agent
main_agent = Agent(
    name="assistant",
    model="gemini-2.5-flash",
    tools=[search_tool, code_tool, custom_tool],
)
```

**Important**: AgentTool is designed for single-tool constraint workaround only.
- OK: Wrapping Google Search agent
- OK: Wrapping Code Execution agent
- NOT OK: Wrapping complex agents with multiple tools (may cause bugs)

### Workaround 2: bypass_multi_tools_limit

For Google Search and Vertex AI Search only.

```python
from google.adk.tools import GoogleSearchTool

search = GoogleSearchTool(bypass_multi_tools_limit=True)

agent = Agent(
    model="gemini-2.0-flash",
    tools=[search, other_tool],  # Now works together
)
```

### Sub-Agent Restrictions

Built-in tools cannot be used within sub-agents, except:
- `GoogleSearchTool` with `bypass_multi_tools_limit=True`
- `VertexAiSearchTool` with `bypass_multi_tools_limit=True`

---

## Summary Table

| Tool | Category | Single-Tool Constraint | Version |
|------|----------|----------------------|---------|
| Google Search | Gemini API | Yes | v0.1.0+ |
| Code Execution | Gemini API | Yes | v0.1.0+ |
| Computer Use | Gemini API | No | v1.17.0+ |
| BigQuery | Google Cloud | No | v1.1.0+ |
| Vertex AI Search | Google Cloud | Yes | v0.1.0+ |
| Vertex AI RAG Engine | Google Cloud | Yes | v0.1.0+ |
| Spanner | Google Cloud | No | v1.11.0+ |
| GitHub (MCP) | Third-Party | No | - |
| Notion (MCP) | Third-Party | No | - |
| Stripe (MCP) | Third-Party | No | - |

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Tools Overview | https://google.github.io/adk-docs/tools/ |
| Gemini API Tools | https://google.github.io/adk-docs/tools/gemini-api/ |
| Google Search | https://google.github.io/adk-docs/tools/gemini-api/google-search/ |
| Code Execution | https://google.github.io/adk-docs/tools/gemini-api/code-execution/ |
| Computer Use | https://google.github.io/adk-docs/tools/gemini-api/computer-use/ |
| Google Cloud Tools | https://google.github.io/adk-docs/tools/google-cloud/ |
| BigQuery | https://google.github.io/adk-docs/tools/google-cloud/bigquery/ |
| Vertex AI Search | https://google.github.io/adk-docs/tools/google-cloud/vertex-ai-search/ |
| Spanner | https://google.github.io/adk-docs/tools/google-cloud/spanner/ |
| Third-Party Tools | https://google.github.io/adk-docs/tools/third-party/ |
| GitHub | https://google.github.io/adk-docs/tools/third-party/github/ |
| Notion | https://google.github.io/adk-docs/tools/third-party/notion/ |
| Stripe | https://google.github.io/adk-docs/tools/third-party/stripe/ |
| Tool Limitations | https://google.github.io/adk-docs/tools/limitations/ |
