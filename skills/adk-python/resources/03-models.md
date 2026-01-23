# Model Configuration

> **Official Docs**: https://google.github.io/adk-docs/agents/models/

ADK supports multiple model providers through native integration and LiteLLM wrapper.

## Overview

| Provider | Integration | Features |
|----------|-------------|----------|
| Gemini (Google AI Studio) | Native | Full feature support |
| Gemini (Vertex AI) | Native | Enterprise features, VPC |
| Claude (Anthropic) | LiteLLM | Via API or Vertex AI |
| Ollama | LiteLLM | Local models |
| vLLM | LiteLLM | Self-hosted inference |
| Multi-provider | LiteLLM | OpenAI, Azure, etc. |

---

## 1. Gemini Models (Google AI Studio)

> **Docs**: https://google.github.io/adk-docs/agents/models/google-gemini/

Native integration with Google AI Studio. Requires `GOOGLE_API_KEY`.

### Environment Setup

```bash
export GOOGLE_API_KEY=your_api_key
```

### Basic Usage

```python
from google.adk.agents import Agent

agent = Agent(
    name="my_agent",
    model="gemini-2.5-flash",  # or "gemini-3-flash-preview" for latest
    instruction="Your instructions here.",
)
```

### Available Models

> **Model List**: https://ai.google.dev/gemini-api/docs/models

#### Gemini 3 (Latest)

| Model ID | Characteristics | Use Case |
|----------|-----------------|----------|
| `gemini-3-pro-preview` | Most intelligent, best reasoning | Complex analysis, coding, agentic |
| `gemini-3-pro-image-preview` | Image generation + text | Visual content creation |
| `gemini-3-flash-preview` | Balanced speed & intelligence | **General use (recommended)** |

#### Gemini 2.5 (Stable)

| Model ID | Characteristics | Use Case |
|----------|-----------------|----------|
| `gemini-2.5-flash` | Speed/cost balance, 1M context | **General RAG, agentic** |
| `gemini-2.5-flash-lite` | Lowest cost, ultra-fast | High throughput, cost optimization |
| `gemini-2.5-pro` | Advanced thinking, complex reasoning | Math, coding, STEM |

#### Gemini 2.5 Specialized

| Model ID | Characteristics | Use Case |
|----------|-----------------|----------|
| `gemini-2.5-flash-image` | Native image generation | Image + text output |
| `gemini-2.5-flash-native-audio-preview-12-2025` | Live API, audio I/O | Real-time voice, streaming |
| `gemini-2.5-flash-preview-tts` | Text-to-speech | Audio generation |
| `gemini-2.5-pro-preview-tts` | High-quality TTS | Premium audio generation |

#### Model Capabilities Comparison

| Capability | 3 Pro | 3 Flash | 2.5 Flash | 2.5 Pro |
|------------|-------|---------|-----------|---------|
| Input Context | 1M | 1M | 1M | 1M |
| Output Tokens | 65K | 65K | 65K | 65K |
| Function Calling | ✓ | ✓ | ✓ | ✓ |
| Code Execution | ✓ | ✓ | ✓ | ✓ |
| Search Grounding | ✓ | ✓ | ✓ | ✓ |
| Thinking | ✓ | ✓ | ✓ | ✓ |
| URL Context | ✓ | ✓ | ✓ | ✓ |
| Caching | ✓ | ✓ | ✓ | ✓ |
| Image Gen | Pro-Image only | ✗ | Flash-Image only | ✗ |

### Generation Config

```python
from google.genai import types

agent = Agent(
    name="my_agent",
    model="gemini-2.5-flash",
    generate_content_config=types.GenerateContentConfig(
        temperature=0.7,
        top_p=0.9,
        top_k=40,
        max_output_tokens=8192,
        response_modalities=["TEXT"],
    ),
)
```

---

## 2. Claude via LiteLLM

> **Docs**: https://google.github.io/adk-docs/agents/models/anthropic/

Use Anthropic Claude models through LiteLLM wrapper.

### Environment Setup

```bash
export ANTHROPIC_API_KEY=your_anthropic_api_key
```

### Basic Usage

```python
from google.adk.agents import Agent
from google.adk.models.lite_llm import LiteLlm

agent = Agent(
    name="claude_agent",
    model=LiteLlm(model="claude-sonnet-4-20250514"),
    instruction="Your instructions here.",
)
```

### Available Claude Models

| Model ID | Characteristics |
|----------|-----------------|
| `claude-sonnet-4-20250514` | Balanced performance |
| `claude-opus-4-20250514` | Highest capability |
| `claude-3-5-sonnet-20241022` | Previous generation |
| `claude-3-5-haiku-20241022` | Fast, efficient |

### With Generation Parameters

```python
agent = Agent(
    name="claude_agent",
    model=LiteLlm(
        model="claude-sonnet-4-20250514",
        temperature=0.7,
        max_tokens=4096,
    ),
)
```

---

## 3. Vertex AI Hosted Models

> **Docs**: https://google.github.io/adk-docs/agents/models/vertex/

Enterprise-grade deployment with Vertex AI.

### Environment Setup

```bash
export GOOGLE_CLOUD_PROJECT=your_project_id
export GOOGLE_CLOUD_LOCATION=us-central1
# Authentication via gcloud or service account
gcloud auth application-default login
```

### Gemini on Vertex AI

```python
from google.adk.agents import Agent
from google.adk.models.vertexai import VertexAiLlm

agent = Agent(
    name="vertex_gemini_agent",
    model=VertexAiLlm(model="gemini-2.5-flash"),
    instruction="Your instructions here.",
)
```

### Claude on Vertex AI (Model Garden)

```python
from google.adk.agents import Agent
from google.adk.models.lite_llm import LiteLlm

agent = Agent(
    name="vertex_claude_agent",
    model=LiteLlm(
        model="vertex_ai/claude-sonnet-4@20250514",
        vertex_project="your-project-id",
        vertex_location="us-east5",
    ),
    instruction="Your instructions here.",
)
```

### Vertex AI Model IDs

| Model Type | Model ID Format |
|------------|-----------------|
| Gemini | `gemini-2.5-flash`, `gemini-2.5-pro` |
| Claude (Model Garden) | `vertex_ai/claude-sonnet-4@20250514` |
| Llama (Model Garden) | `vertex_ai/meta/llama-3.1-405b-instruct-maas` |

---

## 4. Apigee AI Gateway

For enterprise API management with Apigee.

### Configuration

```python
from google.adk.agents import Agent
from google.adk.models.lite_llm import LiteLlm

agent = Agent(
    name="apigee_agent",
    model=LiteLlm(
        model="gemini-2.5-flash",
        api_base="https://your-apigee-endpoint.com/v1",
        api_key="your_apigee_api_key",
    ),
    instruction="Your instructions here.",
)
```

### Benefits

- Centralized API management
- Rate limiting and quotas
- Request/response logging
- Security policies

---

## 5. Ollama (Local Models)

> **Docs**: https://google.github.io/adk-docs/agents/models/ollama/

Run models locally with Ollama.

### Prerequisites

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model
ollama pull llama3.1
ollama pull qwen2.5:7b
```

### Configuration

```python
from google.adk.agents import Agent
from google.adk.models.lite_llm import LiteLlm

agent = Agent(
    name="local_agent",
    model=LiteLlm(
        model="ollama/llama3.1",
        api_base="http://localhost:11434",
    ),
    instruction="Your instructions here.",
)
```

### Available Local Models

| Model | Ollama Model ID | Memory |
|-------|-----------------|--------|
| Llama 3.1 8B | `ollama/llama3.1` | ~8GB |
| Llama 3.1 70B | `ollama/llama3.1:70b` | ~40GB |
| Qwen 2.5 7B | `ollama/qwen2.5:7b` | ~5GB |
| Mistral 7B | `ollama/mistral` | ~5GB |
| Gemma 2 9B | `ollama/gemma2:9b` | ~6GB |

### Ollama Server Options

```bash
# Custom host/port
OLLAMA_HOST=0.0.0.0:11434 ollama serve

# GPU memory fraction
OLLAMA_GPU_MEMORY_FRACTION=0.8 ollama serve
```

---

## 6. vLLM

Self-hosted high-performance inference server.

### Server Setup

```bash
# Start vLLM server
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Meta-Llama-3.1-8B-Instruct \
    --port 8000
```

### Configuration

```python
from google.adk.agents import Agent
from google.adk.models.lite_llm import LiteLlm

agent = Agent(
    name="vllm_agent",
    model=LiteLlm(
        model="openai/meta-llama/Meta-Llama-3.1-8B-Instruct",
        api_base="http://localhost:8000/v1",
        api_key="dummy",  # vLLM doesn't require real key
    ),
    instruction="Your instructions here.",
)
```

### vLLM Benefits

- High throughput with continuous batching
- Tensor parallelism for large models
- OpenAI-compatible API

---

## 7. LiteLLM (Multi-Provider)

> **Docs**: https://google.github.io/adk-docs/agents/models/litellm/

Universal wrapper supporting 100+ LLM providers.

### Installation

```bash
pip install google-adk[litellm]
# or
uv add google-adk[litellm]
```

### OpenAI

```python
from google.adk.agents import Agent
from google.adk.models.lite_llm import LiteLlm
import os

os.environ["OPENAI_API_KEY"] = "your_key"

agent = Agent(
    name="openai_agent",
    model=LiteLlm(model="gpt-4o"),
    instruction="Your instructions here.",
)
```

### Azure OpenAI

```python
import os

os.environ["AZURE_API_KEY"] = "your_key"
os.environ["AZURE_API_BASE"] = "https://your-resource.openai.azure.com"
os.environ["AZURE_API_VERSION"] = "2024-02-15-preview"

agent = Agent(
    name="azure_agent",
    model=LiteLlm(model="azure/gpt-4o-deployment"),
    instruction="Your instructions here.",
)
```

### AWS Bedrock

```python
import os

os.environ["AWS_ACCESS_KEY_ID"] = "your_key"
os.environ["AWS_SECRET_ACCESS_KEY"] = "your_secret"
os.environ["AWS_REGION_NAME"] = "us-east-1"

agent = Agent(
    name="bedrock_agent",
    model=LiteLlm(model="bedrock/anthropic.claude-3-sonnet-20240229-v1:0"),
    instruction="Your instructions here.",
)
```

### Supported Providers

| Provider | Model Format | Env Variable |
|----------|-------------|--------------|
| OpenAI | `gpt-4o`, `gpt-4o-mini` | `OPENAI_API_KEY` |
| Azure | `azure/deployment-name` | `AZURE_API_KEY` |
| Anthropic | `claude-sonnet-4-20250514` | `ANTHROPIC_API_KEY` |
| AWS Bedrock | `bedrock/model-id` | AWS credentials |
| Cohere | `cohere/command-r-plus` | `COHERE_API_KEY` |
| Together AI | `together_ai/model-id` | `TOGETHER_API_KEY` |

---

## Model Selection Guide

### By Use Case

| Use Case | Recommended Model |
|----------|-------------------|
| General RAG | `gemini-3-flash-preview` or `gemini-2.5-flash` |
| Complex reasoning | `gemini-3-pro-preview` or `gemini-2.5-pro` |
| Agentic / Vibe coding | `gemini-3-pro-preview` |
| Cost optimization | `gemini-2.5-flash-lite` |
| Image generation | `gemini-3-pro-image-preview` or `gemini-2.5-flash-image` |
| Real-time voice | `gemini-2.5-flash-native-audio-preview-12-2025` |
| Text-to-speech | `gemini-2.5-flash-preview-tts` |
| Local/private | `ollama/llama3.1` |
| Enterprise | Vertex AI models |

### By Priority

| Priority | Recommendation |
|----------|----------------|
| Intelligence | `gemini-3-pro-preview`, `claude-opus-4` |
| Speed | `gemini-3-flash-preview`, `gemini-2.5-flash-lite` |
| Cost | `gemini-2.5-flash-lite`, local models |
| Stability | `gemini-2.5-flash`, `gemini-2.5-pro` (stable) |
| Privacy | Ollama, vLLM (self-hosted) |

---

## Multi-Model Agents

Different models for different sub-agents:

```python
from google.adk.agents import Agent, SequentialAgent
from google.adk.models.lite_llm import LiteLlm

# Fast model for simple tasks
router = Agent(
    name="router",
    model="gemini-2.5-flash-lite",
    instruction="Route queries to appropriate specialist.",
    sub_agents=[specialist1, specialist2],
)

# Powerful model for complex analysis
analyst = Agent(
    name="analyst",
    model="gemini-2.5-pro",
    instruction="Perform deep analysis.",
)

# Cost-effective model for summarization
summarizer = Agent(
    name="summarizer",
    model=LiteLlm(model="gpt-4o-mini"),
    instruction="Summarize the analysis results.",
)
```

---

## Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| API key not found | Set environment variable or pass explicitly |
| Model not available | Check model ID spelling, region availability |
| Rate limit exceeded | Implement retry logic, use multiple keys |
| Context length exceeded | Use model with larger context or chunk input |

### Debug Model Configuration

```python
from google.adk.models.lite_llm import LiteLlm

model = LiteLlm(model="gpt-4o")
print(f"Provider: {model.provider}")
print(f"Model: {model.model}")
```

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Models Overview | https://google.github.io/adk-docs/agents/models/ |
| Google Gemini | https://google.github.io/adk-docs/agents/models/google-gemini/ |
| Anthropic Claude | https://google.github.io/adk-docs/agents/models/anthropic/ |
| Vertex AI | https://google.github.io/adk-docs/agents/models/vertex/ |
| LiteLLM | https://google.github.io/adk-docs/agents/models/litellm/ |
| Ollama | https://google.github.io/adk-docs/agents/models/ollama/ |
| Gemini Model List | https://ai.google.dev/gemini-api/docs/models |
