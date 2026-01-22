# Evaluation

> **Reference**: https://google.github.io/adk-docs/evaluate/

## Overview

Agent evaluation differs from traditional software testing. LLM-based agents exhibit probabilistic behavior, requiring assessment of both **trajectory** (steps taken) and **response quality** (final output).

### Evaluation Targets

| Target | Description |
|--------|-------------|
| **Trajectory** | Which tools were called, in what order |
| **Final Response** | Quality, accuracy, relevance of the answer |

### Evaluation Methods

| Method | File Type | Use Case | Execution |
|--------|-----------|----------|-----------|
| Test Files | `.test.json` | Unit testing during development | `adk web` |
| Evalsets | `.evalset.json` | Integration testing, multi-turn | `adk eval` |
| pytest | `.py` | CI/CD integration | `pytest` |

---

## Evaluation Criteria

> **Reference**: https://google.github.io/adk-docs/evaluate/criteria/

ADK provides 8 built-in evaluation criteria:

### 1. tool_trajectory_avg_score

Compares tool call sequences against expected calls.

```python
# Matching modes
EXACT       # Exact match required
IN_ORDER    # Subset in correct order
ANY_ORDER   # Subset in any order
```

### 2. response_match_score

ROUGE-1 text similarity between agent response and reference.

### 3. final_response_match_v2

LLM-as-judge for semantic equivalence. Passes if meaning matches even with different wording.

### 4. rubric_based_final_response_quality_v1

Custom rubrics for response quality (tone, helpfulness, completeness).

### 5. rubric_based_tool_use_quality_v1

Evaluates tool selection, parameters, and call sequencing.

### 6. hallucinations_v1

Checks if responses are grounded in context. Identifies unsupported claims.

### 7. safety_v1

Evaluates harmlessness and compliance with safety guidelines.

### 8. per_turn_user_simulator_quality_v1

Validates user simulator fidelity to conversation plans in multi-turn testing.

### Score Interpretation

All criteria return scores from **0.0 to 1.0**:

| Score | Interpretation |
|-------|----------------|
| 0.9+ | Excellent |
| 0.7-0.9 | Good |
| 0.5-0.7 | Needs improvement |
| < 0.5 | Problematic |

---

## Evalset Structure

### Basic Structure

```json
{
  "eval_set_id": "my_evalset",
  "name": "My Agent Evaluation",
  "eval_cases": [
    {
      "eval_id": "case_001",
      "conversation": [
        {
          "user_content": "What's the weather in Seoul?",
          "final_response": "Seoul is currently sunny with 25°C.",
          "intermediate_data": {
            "tool_uses": [
              {
                "name": "get_weather",
                "args": {"city": "Seoul"}
              }
            ]
          }
        }
      ]
    }
  ]
}
```

### Multi-turn Conversation

```json
{
  "eval_id": "multi_turn_001",
  "conversation": [
    {
      "user_content": "Hello",
      "final_response": "Hello! How can I help you?"
    },
    {
      "user_content": "Check Seoul weather",
      "intermediate_data": {
        "tool_uses": [{"name": "get_weather"}]
      },
      "final_response": "Seoul is 25°C and sunny."
    }
  ]
}
```

### File Naming Convention

**Important**: `eval_set_id` must match filename (excluding `.evalset.json`).

| Filename | eval_set_id |
|----------|-------------|
| `weather.evalset.json` | `weather` |
| `my_test.evalset.json` | `my_test` |

---

## Tool Usage Evaluation

### Trajectory Matching

```json
{
  "intermediate_data": {
    "tool_uses": [
      {"name": "search_database", "args": {"query": "batteries"}},
      {"name": "format_results"}
    ]
  }
}
```

### Evaluation Config

```json
{
  "criteria": {
    "tool_trajectory_avg_score": 1.0,
    "rubric_based_tool_use_quality_v1": {
      "rubric": "Tool calls should be minimal and efficient"
    }
  }
}
```

---

## Response Quality Evaluation

### Reference-based Matching

```json
{
  "criteria": {
    "response_match_score": 0.8,
    "final_response_match_v2": {
      "threshold": 0.8,
      "judge_model": "gemini-2.5-flash"
    }
  }
}
```

### Custom Rubrics

```json
{
  "criteria": {
    "rubric_based_final_response_quality_v1": {
      "rubric": "Response should be helpful, accurate, and in Korean."
    }
  }
}
```

---

## User Simulation

> **Reference**: https://google.github.io/adk-docs/evaluate/user-sim/

Dynamically generates user prompts for multi-turn conversation testing.

### Conversation Scenario

```json
{
  "user_simulator": {
    "starting_prompt": "What can you do?",
    "conversation_plan": "Ask about exhibition schedule, then inquire about registration process."
  }
}
```

### User Simulator Config

```python
from google.adk.evaluate import EvalConfig, UserSimulatorConfig

config = EvalConfig(
    user_simulator_config=UserSimulatorConfig(
        model="gemini-2.5-flash",
        max_allowed_invocations=10,
    )
)
```

### Configuration Options

| Option | Default | Description |
|--------|---------|-------------|
| `model` | gemini-2.5-flash | LLM for generating user prompts |
| `max_allowed_invocations` | 20 | Maximum conversation turns |
| `thinking_config` | enabled | Controls extended reasoning |

### Use Cases

- Testing conversational flow without predefined inputs
- Evaluating agent behavior with dynamic user responses
- Stress testing with varied conversation paths

---

## adk eval Command

### Basic Usage

```bash
# Run all cases in evalset
adk eval <AGENT_PATH> <EVALSET_PATH>

# Example
adk eval my_agent/ my_agent/weather.evalset.json
```

### Options

```bash
adk eval my_agent/ evalset.json \
    --config_file_path=config.json \
    --print_detailed_results
```

| Option | Description |
|--------|-------------|
| `--config_file_path` | Path to evaluation config JSON |
| `--print_detailed_results` | Show detailed per-case results |

### Running Specific Cases

```bash
# Run specific eval cases by ID
adk eval my_agent/ evalset.json:case_001,case_002,case_003
```

---

## ADK Web Evaluation

### Setup

Place evalset files inside agent directory:

```
backend/
├── my_agent/
│   ├── agent.py
│   ├── tools.py
│   └── my_agent.evalset.json  # Same name as eval_set_id
```

### Running

```bash
# Start Dev UI with agent reload
adk web --reload_agents backend/

# Access Eval tab in browser
```

### Features

- Interactive case creation
- Real-time evaluation execution
- Debugging failed cases
- Visual trajectory inspection

---

## pytest Integration

```python
import pytest
from google.adk.evaluate import EvalRunner, EvalConfig

@pytest.fixture
def eval_runner(my_agent):
    return EvalRunner(
        agent=my_agent,
        config=EvalConfig(
            criteria={
                "tool_trajectory_avg_score": 1.0,
                "response_match_score": 0.8
            }
        )
    )

def test_weather_query(eval_runner):
    result = eval_runner.run_case({
        "user_content": "Seoul weather?",
        "intermediate_data": {
            "tool_uses": [{"name": "get_weather"}]
        },
        "final_response": "Seoul is sunny."
    })

    assert result.scores["tool_trajectory_avg_score"] >= 0.9
    assert result.scores["response_match_score"] >= 0.7
```

### CI/CD Integration

```yaml
# .github/workflows/test.yml
- name: Run Agent Evaluations
  run: |
    adk eval backend/my_agent/ backend/my_agent/test.evalset.json \
        --print_detailed_results
```

---

## Best Practices

### 1. Start with Critical Paths

Focus evaluations on high-impact user journeys first.

### 2. Use Appropriate Criteria

| Use Case | Recommended Criteria |
|----------|---------------------|
| Tool-heavy agents | `tool_trajectory_avg_score`, `rubric_based_tool_use_quality_v1` |
| Conversational | `final_response_match_v2`, `rubric_based_final_response_quality_v1` |
| Safety-critical | `hallucinations_v1`, `safety_v1` |

### 3. Iterative Refinement

```
1. Write initial evalset
2. Run evaluation
3. Analyze failures
4. Improve agent prompts/tools
5. Update evalset with edge cases
6. Repeat
```

### 4. Combine Methods

- **Development**: `adk web` for interactive testing
- **Pre-commit**: pytest for quick validation
- **CI/CD**: `adk eval` for comprehensive checks

---

## Official Documentation

| Topic | URL |
|-------|-----|
| Evaluation Overview | https://google.github.io/adk-docs/evaluate/ |
| Evaluation Criteria | https://google.github.io/adk-docs/evaluate/criteria/ |
| User Simulation | https://google.github.io/adk-docs/evaluate/user-sim/ |
