# Agents

> **Reference**: https://google.github.io/adk-docs/agents/

## Overview

ADK provides three main agent categories for building AI applications.

| Category | Description | Use Case |
|----------|-------------|----------|
| **LLM Agent** | LLM-based dynamic decision making | Conversational AI, tool selection |
| **Workflow Agent** | Deterministic flow control without LLM | Pipelines, orchestration |
| **Custom Agent** | Arbitrary logic via BaseAgent inheritance | Complex conditional flows |

---

## LLM Agents (Agent)

> **Reference**: https://google.github.io/adk-docs/agents/llm-agents/

The `Agent` class (alias: `LlmAgent`) uses LLM for dynamic reasoning and decision making.

### Basic Structure

```python
from google.adk.agents import Agent

agent = Agent(
    name="my_agent",
    model="gemini-2.5-flash",
    description="Agent description (used when delegating)",
    instruction="System prompt (agent behavior guidelines)",
    tools=[tool1, tool2],
    sub_agents=[child_agent1, child_agent2],
    output_key="result",
)
```

### Key Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `name` | Yes | Unique identifier for the agent |
| `model` | Yes | Gemini model ID (e.g., `gemini-2.5-flash`) |
| `description` | No | Role description (used for delegation) |
| `instruction` | No | System prompt defining behavior |
| `tools` | No | List of available tools |
| `sub_agents` | No | List of child agents |
| `output_key` | No | Key to store response in `session.state` |

### Example: Weather Agent

```python
from google.adk.agents import Agent

def get_weather(city: str) -> str:
    """Get current weather for a city.

    Args:
        city: City name to get weather for

    Returns:
        Weather information for the city
    """
    return f"{city}: Sunny, 25C"

weather_agent = Agent(
    name="weather_agent",
    model="gemini-2.5-flash",
    description="Provides weather information for any city",
    instruction="""You are a weather assistant.
    - Use get_weather tool to fetch weather data
    - Provide friendly, helpful responses
    - Include temperature and conditions""",
    tools=[get_weather],
)
```

---

## Workflow Agents

> **Reference**: https://google.github.io/adk-docs/agents/workflow-agents/

Workflow Agents control sub-agent execution **deterministically without LLM**.

| Type | Class | Description | Use Case |
|------|-------|-------------|----------|
| Sequential | `SequentialAgent` | Execute in order | Step-by-step pipelines |
| Loop | `LoopAgent` | Repeat until condition | Iterative refinement |
| Parallel | `ParallelAgent` | Execute concurrently | Independent tasks |

### SequentialAgent

Executes sub-agents in defined order.

```python
from google.adk.agents import Agent, SequentialAgent

# Step 1: Write draft
writer = Agent(
    name="writer",
    model="gemini-2.5-flash",
    instruction="Write a draft about the topic.",
    output_key="draft",
)

# Step 2: Review draft
reviewer = Agent(
    name="reviewer",
    model="gemini-2.5-flash",
    instruction="Review this draft: {draft}",
    output_key="review",
)

# Step 3: Final edit
editor = Agent(
    name="editor",
    model="gemini-2.5-flash",
    instruction="Edit based on review ({review}): {draft}",
    output_key="final",
)

# Pipeline
pipeline = SequentialAgent(
    name="writing_pipeline",
    sub_agents=[writer, reviewer, editor],
)
```

**State Flow:**
```
writer (output_key="draft")
    |
    v
session.state["draft"] = "Written content..."
    |
    v
reviewer (instruction references {draft})
    |
    v
session.state["review"] = "Review feedback..."
    |
    v
editor (instruction references {draft}, {review})
```

### LoopAgent

Repeats sub-agents until termination condition.

**Termination Methods:**

| Method | Description |
|--------|-------------|
| `max_iterations` | Stop after N iterations |
| `escalate = True` | Tool signals loop exit |

```python
from google.adk.agents import Agent, LoopAgent
from google.adk.tools import ToolContext

def exit_loop(tool_context: ToolContext) -> dict:
    """Exit the loop when quality is sufficient."""
    tool_context.actions.escalate = True  # Key!
    return {"status": "completed"}

critic = Agent(
    name="critic",
    model="gemini-2.5-flash",
    instruction="""Evaluate the draft.
    - If quality is good, call exit_loop
    - Otherwise, provide feedback""",
    tools=[exit_loop],
    output_key="feedback",
)

refiner = Agent(
    name="refiner",
    model="gemini-2.5-flash",
    instruction="Improve based on feedback ({feedback}): {draft}",
    output_key="draft",
)

refinement_loop = LoopAgent(
    name="refinement_loop",
    sub_agents=[critic, refiner],
    max_iterations=5,  # Safety limit
)
```

### ParallelAgent

Executes sub-agents concurrently for independent tasks.

```python
from google.adk.agents import Agent, ParallelAgent, SequentialAgent

# Independent researchers
researcher_a = Agent(
    name="researcher_energy",
    model="gemini-2.5-flash",
    instruction="Research renewable energy.",
    output_key="energy_research",
)

researcher_b = Agent(
    name="researcher_ev",
    model="gemini-2.5-flash",
    instruction="Research electric vehicles.",
    output_key="ev_research",
)

# Parallel execution
parallel_research = ParallelAgent(
    name="parallel_research",
    sub_agents=[researcher_a, researcher_b],
)

# Synthesize results
synthesizer = Agent(
    name="synthesizer",
    model="gemini-2.5-flash",
    instruction="""Synthesize findings:
    - Energy: {energy_research}
    - EV: {ev_research}""",
    output_key="final_report",
)

# Fan-Out/Gather Pattern
research_pipeline = SequentialAgent(
    name="research_pipeline",
    sub_agents=[parallel_research, synthesizer],
)
```

**Important Notes:**
- No state sharing between parallel agents (race condition risk)
- Completion order is not guaranteed
- Use unique `output_key` for each parallel agent

---

## Custom Agents

> **Reference**: https://google.github.io/adk-docs/agents/custom-agents/

Inherit `BaseAgent` to implement arbitrary orchestration logic.

### Basic Structure

```python
from google.adk.agents import BaseAgent
from google.adk.events import Event
from typing import AsyncGenerator

class MyCustomAgent(BaseAgent):
    def __init__(self, name: str, sub_agents: list):
        super().__init__(name=name, sub_agents=sub_agents)
        self.analyzer = sub_agents[0]
        self.executor = sub_agents[1]

    async def _run_async_impl(
        self, ctx
    ) -> AsyncGenerator[Event, None]:
        # Step 1: Run analyzer
        async for event in self.analyzer.run_async(ctx):
            yield event

        # Step 2: Check condition
        result = ctx.session.state.get("analysis_result")

        if result == "success":
            # Step 3: Run executor only on success
            async for event in self.executor.run_async(ctx):
                yield event
```

### Key Methods

| Method | Description |
|--------|-------------|
| `_run_async_impl(ctx)` | Main execution logic (returns AsyncGenerator) |
| `ctx.session.state` | Shared state between agents |
| `run_async(ctx)` | Execute child agent |

### Conditional Routing Example

```python
class RouterAgent(BaseAgent):
    def __init__(
        self,
        name: str,
        weather_agent,
        calculator_agent,
        default_agent
    ):
        super().__init__(
            name=name,
            sub_agents=[weather_agent, calculator_agent, default_agent]
        )
        self.weather = weather_agent
        self.calculator = calculator_agent
        self.default = default_agent

    async def _run_async_impl(self, ctx):
        user_message = ctx.session.state.get("user_message", "")

        if "weather" in user_message.lower():
            target = self.weather
        elif "calculate" in user_message.lower():
            target = self.calculator
        else:
            target = self.default

        async for event in target.run_async(ctx):
            yield event
```

---

## Multi-Agent Systems

> **Reference**: https://google.github.io/adk-docs/agents/multi-agents/

### Delegation via sub_agents

LLM automatically delegates to appropriate child agent.

```python
from google.adk.agents import Agent

greeting_agent = Agent(
    name="greeter",
    model="gemini-2.5-flash",
    description="Handles greetings and small talk",
    instruction="Respond warmly to greetings.",
)

weather_agent = Agent(
    name="weather_expert",
    model="gemini-2.5-flash",
    description="Provides weather information",
    instruction="Answer weather questions.",
    tools=[get_weather],
)

# Coordinator (root agent)
root_agent = Agent(
    name="coordinator",
    model="gemini-2.5-flash",
    instruction="Route requests to appropriate expert.",
    sub_agents=[greeting_agent, weather_agent],
)
```

**Internal Mechanism:**
1. LLM generates `transfer_to_agent(agent_name='weather_expert')`
2. ADK AutoFlow routes to target agent

### AgentTool (Explicit Invocation)

Wrap agent as a tool for explicit calling.

```python
from google.adk.tools import AgentTool

# Wrap agent as tool
weather_tool = AgentTool(weather_agent)

coordinator = Agent(
    name="coordinator",
    model="gemini-2.5-flash",
    tools=[weather_tool],  # Include AgentTool
    instruction="Use tools as needed.",
)
```

**Comparison:**

| Method | Behavior | Control |
|--------|----------|---------|
| `sub_agents` + delegation | LLM auto-routes | Implicit |
| `AgentTool` | Explicit tool call | Explicit |

**AgentTool Limitations:**

| Use Case | Suitable | Note |
|----------|----------|------|
| Tool-only agents (Google Search) | Yes | Recommended |
| Simple agents without tools | Yes | Works |
| **Agents with tools** | No | Bug (GitHub #4159) |

---

## Agent Config

> **Reference**: https://google.github.io/adk-docs/agents/config/

### YAML Definition

Define agents declaratively in YAML.

```yaml
# agent_config.yaml
name: my_agent
model: gemini-2.5-flash
description: A helpful assistant
instruction: |
  You are a helpful assistant.
  - Answer questions accurately
  - Be concise and friendly
tools:
  - get_weather
  - search_web
sub_agents:
  - calculator_agent
```

### Loading YAML Config

```python
from google.adk.agents import Agent
import yaml

with open("agent_config.yaml") as f:
    config = yaml.safe_load(f)

agent = Agent(**config)
```

### Visual Builder

ADK provides a visual interface for building agent configurations.

```bash
# Launch Visual Builder
adk web
```

Features:
- Drag-and-drop agent composition
- Visual workflow design
- Real-time testing
- Export to Python/YAML

---

## Best Practices

### 1. Clear description for delegation

```python
# Bad
agent = Agent(description="Helps")

# Good
agent = Agent(
    description="Provides weather forecasts including "
                "temperature, humidity, and precipitation"
)
```

### 2. Specific instructions

```python
agent = Agent(
    instruction="""You are a customer support agent.

## Role
- Answer product questions
- Help with technical issues

## Rules
1. Always be polite
2. Verify uncertain information
3. Escalate when unable to resolve""",
)
```

### 3. Separate concerns with sub_agents

```python
# Bad: One agent with everything
agent = Agent(tools=[search, calc, translate, ...])

# Good: Specialized agents
search_agent = Agent(name="searcher", tools=[search])
calc_agent = Agent(name="calculator", tools=[calculate])

coordinator = Agent(
    name="coordinator",
    sub_agents=[search_agent, calc_agent],
)
```

### 4. Always set max_iterations for LoopAgent

```python
# Bad: Infinite loop risk
loop = LoopAgent(sub_agents=[refiner])

# Good: Safety limit
loop = LoopAgent(
    sub_agents=[refiner],
    max_iterations=10,
)
```

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Agents Overview | https://google.github.io/adk-docs/agents/ |
| LLM Agents | https://google.github.io/adk-docs/agents/llm-agents/ |
| Workflow Agents | https://google.github.io/adk-docs/agents/workflow-agents/ |
| Custom Agents | https://google.github.io/adk-docs/agents/custom-agents/ |
| Multi-Agents | https://google.github.io/adk-docs/agents/multi-agents/ |
| Agent Config | https://google.github.io/adk-docs/agents/config/ |
