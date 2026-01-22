# Getting Started with ADK Python

> **Official Docs**:
> - [Installation](https://google.github.io/adk-docs/get-started/installation/)
> - [Python Quickstart](https://google.github.io/adk-docs/get-started/python/)

## Installation

### Using pip

```bash
pip install google-adk
```

### Using uv (recommended)

```bash
uv add google-adk
```

### Verify Installation

```bash
adk --version
```

---

## Environment Variables

Create a `.env` file in your project root:

```env
GOOGLE_API_KEY=your_gemini_api_key
```

| Variable | Required | Description |
|----------|----------|-------------|
| `GOOGLE_API_KEY` | Yes | Gemini API key from [Google AI Studio](https://aistudio.google.com/) |
| `GOOGLE_CLOUD_PROJECT` | No | GCP project ID (for Vertex AI deployment) |
| `GOOGLE_CLOUD_LOCATION` | No | GCP region (default: `us-central1`) |

---

## Project Structure

ADK CLI requires a specific project structure to work correctly:

```
my_agent/
├── __init__.py          # (optional) re-export root_agent
├── agent.py             # root_agent = Agent(...) REQUIRED
├── tools.py             # (optional) custom tools
├── prompts.py           # (optional) system prompts
└── .env                 # environment variables
```

### Critical: root_agent Requirement

The `agent.py` file **must** export a variable named `root_agent`:

```python
# agent.py

from google.adk.agents import Agent

# Option 1: Direct assignment
root_agent = Agent(
    name="my_agent",
    model="gemini-2.5-flash",
    instruction="Your instructions here",
)

# Option 2: Alias for existing agent
my_custom_agent = Agent(...)
root_agent = my_custom_agent  # ADK CLI compatible
```

### Common Mistakes

```python
# BAD - ADK CLI won't find this
my_agent = Agent(name="my_agent", ...)

# GOOD - ADK CLI will recognize this
root_agent = Agent(name="my_agent", ...)
```

---

## ADK CLI Commands

| Command | Description | Example |
|---------|-------------|---------|
| `adk create <name>` | Create Python agent project | `adk create my_agent` |
| `adk create --type=config <name>` | Create YAML agent project | `adk create --type=config my_agent` |
| `adk web <dir>` | Launch Dev UI (browser) | `adk web backend/agents` |
| `adk run <agent>` | Run agent in terminal | `adk run my_agent` |
| `adk api_server <dir>` | Start REST API server | `adk api_server backend/agents` |
| `adk eval` | Run agent evaluation | `adk eval --evalset test.json` |

### adk create

Scaffold a new agent project:

```bash
# Python-based agent (default)
adk create my_agent

# YAML-based agent (for Visual Builder)
adk create --type=config my_agent
```

Generated structure:

```
my_agent/
├── __init__.py
├── agent.py          # Python: root_agent definition
├── root_agent.yaml   # YAML: agent configuration
└── .env
```

### adk web

Launch the Development UI for interactive testing:

```bash
# Single agent
adk web my_agent

# Multiple agents (select in UI)
adk web backend/agents

# Custom port
adk web --port 9100 backend/agents
```

**Features:**
- Interactive chat interface
- View agent events and tool calls
- Debug session state
- Visual Builder integration (+ button)

### adk run

Run agent in terminal for quick testing:

```bash
adk run my_agent

# With specific session
adk run my_agent --session_id test_session
```

### adk api_server

Start a REST API server for production or integration:

```bash
# Basic usage
adk api_server backend/agents

# Custom host and port
adk api_server backend/agents --host 0.0.0.0 --port 9101

# With specific agent directory
adk api_server backend/agents/rag_agent
```

**API Endpoints:**
- `POST /run` - Execute agent
- `POST /run_sse` - Execute with Server-Sent Events
- `GET /list-apps` - List available agents

---

## Minimal Example

### 1. Create Project

```bash
mkdir my_first_agent
cd my_first_agent
```

### 2. Create agent.py

```python
# agent.py
from google.adk.agents import Agent

def get_weather(city: str) -> str:
    """Get current weather for a city.

    Args:
        city: Name of the city to get weather for.

    Returns:
        Weather information for the specified city.
    """
    return f"Weather in {city}: Sunny, 25C"

root_agent = Agent(
    name="weather_agent",
    model="gemini-2.5-flash",
    description="Provides weather information for cities worldwide.",
    instruction="You are a helpful weather assistant. Use the get_weather tool to answer weather questions.",
    tools=[get_weather],
)
```

### 3. Create .env

```env
GOOGLE_API_KEY=your_api_key_here
```

### 4. Run

```bash
# Terminal mode
adk run .

# Dev UI mode
adk web .
```

---

## Advanced Setup Notes

### Multi-Agent Directory Structure

For projects with multiple agents:

```
backend/
├── agents/
│   ├── rag_agent/
│   │   ├── __init__.py
│   │   ├── agent.py       # root_agent
│   │   └── tools.py
│   ├── search_agent/
│   │   ├── __init__.py
│   │   └── agent.py       # root_agent
│   └── __init__.py
└── adk_shared/            # shared utilities
    ├── __init__.py
    └── search_tools.py
```

Run with parent directory to auto-discover agents:

```bash
adk web backend/agents  # All agents selectable in UI
```

### Path Conventions

- Use `/` not `.` in paths: `backend/agents` not `backend.agents`
- Paths are relative to current working directory
- Agent directory must contain `agent.py` with `root_agent`

### Virtual Environment Integration

```bash
# Using uv (recommended)
uv sync
uv run adk web backend/agents

# Using venv
python -m venv .venv
source .venv/bin/activate
pip install google-adk
adk web backend/agents
```

### IDE Integration

For VS Code, add to `.vscode/settings.json`:

```json
{
  "python.analysis.extraPaths": ["backend"],
  "python.envFile": "${workspaceFolder}/.env"
}
```

### Running Behind Proxy

```bash
export HTTP_PROXY=http://proxy:8080
export HTTPS_PROXY=http://proxy:8080
adk web backend/agents
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `root_agent not found` | Ensure `agent.py` exports `root_agent` variable |
| `GOOGLE_API_KEY not set` | Create `.env` file in agent directory or parent |
| `Module not found` | Check virtual environment is activated |
| `Port already in use` | Use `--port` flag with different port |

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Installation | https://google.github.io/adk-docs/get-started/installation/ |
| Python Quickstart | https://google.github.io/adk-docs/get-started/python/ |
| ADK CLI Reference | https://google.github.io/adk-docs/runtime/cli/ |
| Visual Builder | https://google.github.io/adk-docs/visual-builder/ |
| GitHub Repository | https://github.com/google/adk-python |
