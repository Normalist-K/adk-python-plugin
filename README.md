# ADK Python Plugin for Claude Code

Google Agent Development Kit (ADK) Python development guide plugin for Claude Code.

## Features

- Comprehensive ADK Python development guide
- 22 reference documents covering all ADK concepts
- 30+ official sample agents with real-world examples
- Keywords: `google-adk`, `adk-python`, `agent`, `gemini`, `multi-agent`, `tool`, `workflow`

## Installation

```bash
# Install using Claude Code CLI
claude plugins install Normalist-K/adk-python-plugin
```

## What's Included

### SKILL.md

Quick reference covering:
- ADK 5 core concepts (Agent, Tool, Session, Callback, Memory)
- Agent types (LlmAgent, Sequential, Parallel, Loop, Custom)
- Quick start guide
- Project structure

### Reference Documents (22 files)

| Category | Topics |
|----------|--------|
| **Build Agents** | Getting Started, Agents, Models, Built-in Tools, Custom Tools |
| **Run Agents** | Runtime, Deployment, Observability, Evaluation, Safety |
| **Components** | Context, Sessions, Memory, Callbacks, Artifacts, Events |
| **Advanced** | Apps, Plugins, MCP, A2A Protocol, Streaming, Grounding |

### Sample Agents (30+ examples)

Official ADK sample agents organized by category:

| Category | Examples |
|----------|----------|
| **Research** | academic-research, fomc-research, deep-search |
| **E-commerce** | brand-search-optimization, personalized-shopping |
| **Customer Service** | customer-service, travel-concierge |
| **Finance** | currency-agent, financial-advisor, auto-insurance-agent |
| **Data** | data-engineering, data-science, google-trends-agent |
| **Healthcare** | medical-pre-authorization |
| **Safety** | llm-auditor, safety-plugins, ai-security-agent |
| **Media** | short-movie-agents, image-scoring |
| **Marketing** | marketing-agency, brand-aligner |
| **DevOps** | software-bug-assistant, incident-management |
| **Workflow** | hierarchical-workflow-automation, parallel_task_decomposition_execution |

## Usage

After installation, the skill activates automatically when working with:

- Google ADK agent development
- Gemini model integration
- Workflow agents (Sequential, Loop, Parallel)
- Multi-agent systems
- Custom tools and MCP integration
- Agent deployment and evaluation
- Streaming and A2A protocol

### Trigger Keywords

The skill triggers on keywords like:
- `google-adk`, `adk-python`, `adk`
- `agent`, `gemini`, `multi-agent`
- `workflow agent`, `sequential`, `parallel`, `loop`
- `tool`, `mcp`, `a2a`, `grounding`

## Quick Start with ADK

```python
from google.adk.agents import Agent

def get_weather(city: str) -> str:
    """Get weather for a city."""
    return f"{city}: Sunny, 25°C"

agent = Agent(
    name="weather_agent",
    model="gemini-2.5-flash",
    description="Provides weather information",
    instruction="Answer weather questions using the tool.",
    tools=[get_weather],
)
```

## Official Documentation

- [ADK Docs](https://google.github.io/adk-docs/)
- [ADK Python GitHub](https://github.com/google/adk-python)
- [ADK Samples](https://github.com/google/adk-samples)

## License

MIT License
