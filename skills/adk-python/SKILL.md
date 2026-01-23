---
name: adk-python
description: Google Agent Development Kit (ADK) Python development guide. Build, evaluate, and deploy AI agents with Gemini models. Covers agents (LLM, Workflow, Custom, Multi-agent), tools (built-in, custom, MCP), runtime, deployment, observability, evaluation, sessions, memory, callbacks, artifacts, events, streaming, A2A protocol, and grounding. Keywords: google-adk, adk-python, agent, gemini, tool, workflow, multi-agent, deployment, evaluation, streaming, a2a, grounding
---

# Google ADK Python Guide

## Purpose

Comprehensive guide for building, evaluating, and deploying AI agents using Google Agent Development Kit (ADK) with Python.

## When to Use

This skill activates when working with:
- Google ADK agent development
- Gemini model integration
- Workflow agents (Sequential, Loop, Parallel)
- Multi-agent systems
- Custom tools and MCP integration
- Agent deployment and evaluation
- Streaming and A2A protocol

---

## ADK Overview

Google ADK is a **framework for building, evaluating, and deploying AI agents**.

### 5 Core Concepts

| Concept | Description |
|---------|-------------|
| **Agent** | Basic unit for specific tasks (LLM-based or workflow-based) |
| **Tool** | External API interaction, code execution, etc. |
| **Session & State** | Conversation context and agent working memory |
| **Callback** | Custom code execution at specific points during agent run |
| **Memory** | Long-term user information recall across sessions |

### Agent Types

| Type | Description | Use Case |
|------|-------------|----------|
| `Agent` (LlmAgent) | LLM-based dynamic decision making | General conversation, tool selection |
| `SequentialAgent` | Sequential execution | Pipeline, step-by-step processing |
| `ParallelAgent` | Parallel execution | Concurrent search, independent tasks |
| `LoopAgent` | Iterative execution | Retry, quality improvement |
| `CustomAgent` | BaseAgent inheritance | Complex conditional logic |

### Multi-Agent Patterns (핵심)

> **중요**: Multi-agent 시스템 구현 시 반드시 참고해야 할 8가지 디자인 패턴입니다.
> 상세 내용: [23-multi-agent-patterns.md](resources/23-multi-agent-patterns.md)

| Pattern | Agent Type | Use Case |
|---------|------------|----------|
| Sequential Pipeline | `SequentialAgent` | 단계별 파이프라인 |
| Coordinator/Dispatcher | `Agent` + sub_agents | 의도 분석 후 라우팅 |
| Parallel Fan-Out/Gather | `ParallelAgent` | 병렬 처리 후 통합 |
| Hierarchical Decomposition | `AgentTool` | 서브태스크 위임 |
| Generator/Critic | `LoopAgent` | 생성-검증 반복 |
| Iterative Refinement | `LoopAgent` | 점진적 품질 개선 |
| Human-in-the-Loop | Custom Tool | 인간 승인 필요 |
| Composite | Mixed | 패턴 조합 |

---

## Quick Start

### 1. Installation

```bash
pip install google-adk
# or
uv add google-adk
```

### 2. Environment Variables

```env
GOOGLE_API_KEY=your_gemini_api_key
```

### 3. Basic Agent

```python
from google.adk.agents import Agent

def get_weather(city: str) -> str:
    """Get weather for a city.

    Args:
        city: City name to get weather for

    Returns:
        Weather information for the city
    """
    return f"{city}: Sunny, 25°C"

agent = Agent(
    name="weather_agent",
    model="gemini-2.5-flash",
    description="Provides weather information",
    instruction="Answer weather questions using the tool.",
    tools=[get_weather],
)
```

### 4. Project Structure

```
my_agent/
├── __init__.py
├── agent.py          # root_agent = Agent(...) required
├── tools.py          # Tool definitions
└── .env              # Environment variables
```

**Important**: Variable name must be `root_agent` in `agent.py`.

### 5. CLI Commands

```bash
adk web my_agent              # Dev UI
adk run my_agent              # Terminal
adk api_server backend/agents # REST API
```

---

## Reference Files

Detailed information in reference files:

### Build Agents

#### [01-getting-started.md](resources/01-getting-started.md)
- Python installation (pip, uv)
- Environment setup
- Project structure
- CLI commands

#### [02-agents.md](resources/02-agents.md)
- LLM Agents
- Workflow Agents (Sequential, Loop, Parallel)
- Custom Agents (BaseAgent)
- Multi-agent Systems
- Agent Config (YAML)

#### [03-models.md](resources/03-models.md)
- Gemini models
- Claude (via LiteLLM)
- Vertex AI hosted
- Ollama, vLLM, LiteLLM

#### [04-tools-builtin.md](resources/04-tools-builtin.md)
- Gemini API tools (Code Execution, Google Search)
- Google Cloud tools (BigQuery, Vertex AI Search)
- Third-party tools (GitHub, Notion)

#### [05-tools-custom.md](resources/05-tools-custom.md)
- Function Tools
- MCP Tools
- OpenAPI Tools
- Authentication (OAuth2)

### Run Agents

#### [06-runtime.md](resources/06-runtime.md)
- Web Interface
- Command Line
- API Server
- RunConfig

#### [07-deployment.md](resources/07-deployment.md)
- Agent Engine
- Cloud Run
- GKE

#### [08-observability.md](resources/08-observability.md)
- Logging
- Cloud Trace
- External tools

#### [09-evaluation.md](resources/09-evaluation.md)
- Evaluation Criteria
- User Simulation

#### [10-safety.md](resources/10-safety.md)
- Safety and Security

### Components

#### [11-context.md](resources/11-context.md)
- Context Caching
- Context Compaction

#### [12-sessions.md](resources/12-sessions.md)
- Sessions
- Rewind
- State

#### [13-memory.md](resources/13-memory.md)
- Long-term Memory

#### [14-callbacks.md](resources/14-callbacks.md)
- Callback Types
- Design Patterns

#### [15-artifacts.md](resources/15-artifacts.md)
- Artifacts

#### [16-events.md](resources/16-events.md)
- Events

#### [17-apps.md](resources/17-apps.md)
- Apps
- Technical Overview

#### [18-plugins.md](resources/18-plugins.md)
- Plugins

#### [19-mcp.md](resources/19-mcp.md)
- Model Context Protocol

#### [20-a2a.md](resources/20-a2a.md)
- Agent-to-Agent Protocol

#### [21-streaming.md](resources/21-streaming.md)
- Bidi-streaming (Live API)

#### [22-grounding.md](resources/22-grounding.md)
- Google Search Grounding
- Vertex AI Search Grounding

#### [23-multi-agent-patterns.md](resources/23-multi-agent-patterns.md) ⭐
- **8 Design Patterns for Multi-Agent Systems**
- Sequential, Coordinator, Parallel, Hierarchical
- Generator/Critic, Iterative Refinement
- Human-in-the-Loop, Composite Patterns

---

## Sample Agents (Best Practices)

Official ADK sample agents with 40+ real-world examples.

> **Note**: 아래 샘플들은 [Multi-Agent Patterns](resources/23-multi-agent-patterns.md)를 기반으로 구현되어 있습니다.
> 패턴을 먼저 이해한 후 샘플을 참고하면 더 효과적입니다.

- **Local Path**: [samples/python/agents/](samples/python/agents/)
- **Examples List**: [samples/python/agents/README.md](samples/python/agents/README.md)
- **Latest Version**: https://github.com/google/adk-samples/tree/main/python/agents

### Categories

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
| **Integration** | RAG, antom-payment, order-processing |
| **Workflow** | hierarchical-workflow-automation, parallel_task_decomposition_execution |

> **Note**: For latest examples, check the [GitHub repository](https://github.com/google/adk-samples)

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| ADK Main | https://google.github.io/adk-docs/ |
| Get Started | https://google.github.io/adk-docs/get-started/python/ |
| Agents | https://google.github.io/adk-docs/agents/ |
| Tools | https://google.github.io/adk-docs/tools/ |
| Custom Tools | https://google.github.io/adk-docs/tools-custom/ |
| Runtime | https://google.github.io/adk-docs/runtime/ |
| Deployment | https://google.github.io/adk-docs/deploy/ |
| Observability | https://google.github.io/adk-docs/observability/ |
| Evaluation | https://google.github.io/adk-docs/evaluate/ |
| Context | https://google.github.io/adk-docs/context/ |
| Sessions | https://google.github.io/adk-docs/sessions/ |
| Callbacks | https://google.github.io/adk-docs/callbacks/ |
| Streaming | https://google.github.io/adk-docs/streaming/ |
| A2A | https://google.github.io/adk-docs/a2a/ |
| Grounding | https://google.github.io/adk-docs/grounding/ |
| API Reference | https://google.github.io/adk-docs/api-reference/python/ |
| GitHub | https://github.com/google/adk-python |

---

**Skill Status**: COMPLETE
**Line Count**: < 500 (500-line rule compliant)
**Progressive Disclosure**: Reference files for detailed info
