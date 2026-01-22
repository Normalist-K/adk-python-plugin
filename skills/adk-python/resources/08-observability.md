# Observability

ADK provides comprehensive observability features for monitoring, debugging, and analyzing agent applications. This guide covers built-in logging, Google Cloud integrations, and third-party observability tools.

**Official Docs**: [Observability Overview](https://google.github.io/adk-docs/observability/)

---

## 1. Logging

ADK uses Python's standard `logging` module with hierarchical loggers based on module paths.

**Official Docs**: [Logging](https://google.github.io/adk-docs/observability/logging/)

### Basic Configuration

```python
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(levelname)s - %(name)s - %(message)s'
)
```

### CLI Configuration

```bash
# Using --log_level flag
adk web --log_level DEBUG path/to/agents_dir

# Using -v shortcut for verbose
adk web -v path/to/agents_dir
```

### Log Levels

| Level | Purpose | Use Case |
|-------|---------|----------|
| `DEBUG` | Fine-grained diagnostics | Full LLM prompts, API responses, state transitions |
| `INFO` | Lifecycle events | Agent init, session creation, tool execution |
| `WARNING` | Potential issues | Deprecated features, recoverable errors |
| `ERROR` | Operation failures | Failed API calls, unhandled exceptions |
| `CRITICAL` | Critical failures | System-level failures |

> **Production**: Use `INFO` or `WARNING`. Enable `DEBUG` only for troubleshooting.

### Log Message Structure

```
2025-01-15 10:30:45 - DEBUG - google_adk.models.google_llm - LLM request sent
│                      │       │                           │
Timestamp              Level   Logger Name (module path)   Message
```

---

## 2. Cloud Trace Integration

Cloud Trace enables distributed tracing for ADK applications deployed on Google Cloud.

**Official Docs**: [Cloud Trace](https://google.github.io/adk-docs/observability/cloud-trace/)

### Key Features

- Trace agent interactions continuously
- Debug latency issues and errors
- Visualize with heatmaps, line charts, and waterfall views

### Agent Engine Deployment

**CLI:**
```bash
adk deploy agent_engine \
    --project=$GOOGLE_CLOUD_PROJECT \
    --region=$GOOGLE_CLOUD_LOCATION \
    --staging_bucket=$STAGING_BUCKET \
    --trace_to_cloud \
    $AGENT_PATH
```

**Python SDK:**
```python
from vertexai.preview import reasoning_engines

adk_app = reasoning_engines.AdkApp(
    agent=root_agent,
    enable_tracing=True,
)
```

### Cloud Run Deployment

```bash
adk deploy cloud_run \
    --project=$GOOGLE_CLOUD_PROJECT \
    --region=$GOOGLE_CLOUD_LOCATION \
    --trace_to_cloud \
    $AGENT_PATH
```

### Custom Deployment (FastAPI)

```python
from google.adk.cli.fast_api import get_fast_api_app

app = get_fast_api_app(
    agents_dir=AGENT_DIR,
    web=True,
    trace_to_cloud=True,
)
```

### Using OpenTelemetry Directly

```python
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace import export

provider = TracerProvider()
processor = export.BatchSpanProcessor(
    CloudTraceSpanExporter(project_id="{your-project-id}")
)
provider.add_span_processor(processor)
```

### Span Types

Cloud Trace captures spans for: `invocation`, `agent_run`, `call_llm`, `execute_tool`

---

## 3. BigQuery Agent Analytics

Capture operational events directly into BigQuery tables for analysis.

**Official Docs**: [BigQuery Agent Analytics](https://google.github.io/adk-docs/observability/bigquery-agent-analytics/)

### Features

- Async logging via BigQuery Storage Write API
- Multimodal content logging with GCS offloading
- OpenTelemetry-style distributed tracing support

### Prerequisites

- BigQuery API enabled
- BigQuery dataset for logs
- GCS bucket (optional, for multimodal content)
- IAM roles: `roles/bigquery.jobUser`, `roles/bigquery.dataEditor`, `roles/storage.objectCreator`

### Setup

```python
from google.adk.plugins.bigquery_agent_analytics_plugin import (
    BigQueryAgentAnalyticsPlugin, BigQueryLoggerConfig
)
from google.adk.runners import App

bq_config = BigQueryLoggerConfig(
    enabled=True,
    gcs_bucket_name="your-bucket",
    max_content_length=500 * 1024
)

plugin = BigQueryAgentAnalyticsPlugin(
    project_id="your-project",
    dataset_id="your-dataset",
    config=bq_config
)

app = App(root_agent=agent, plugins=[plugin])
```

### Configuration Options

| Option | Description | Default |
|--------|-------------|---------|
| `batch_size` | Events to batch before writing | 1 |
| `event_allowlist` | Filter events to log | None |
| `event_denylist` | Filter events to exclude | None |
| `content_formatter` | Custom content formatting function | None |
| `log_multi_modal_content` | Enable detailed content logging | False |
| `shutdown_timeout` | Flush duration on exit | 10s |

### Data Schema

| Column | Purpose |
|--------|---------|
| `timestamp` | UTC event time |
| `event_type` | LLM_REQUEST, TOOL_COMPLETED, etc. |
| `session_id` | Groups events in conversations |
| `trace_id` / `span_id` | Distributed tracing |
| `content` | Event-specific JSON payload |
| `content_parts` | Multimodal content references |
| `status` | OK or ERROR |

---

## 4. External Observability Tools

### AgentOps

**Official Docs**: [AgentOps](https://google.github.io/adk-docs/observability/agentops/)

Session replays, metrics, and monitoring with minimal setup.

```bash
pip install -U agentops
```

```python
import agentops

# Set AGENTOPS_API_KEY environment variable
agentops.init()

# Extended configuration
agentops.init(
    api_key=os.getenv("AGENTOPS_API_KEY"),
    trace_name="my-adk-app-trace",
)
```

**Features:**
- Hierarchical spans for agents, LLM calls, and tools
- Prompts, completions, token counts capture
- LLM cost and latency tracking

Get API key: [AgentOps Dashboard](https://app.agentops.ai/settings/projects)

---

### Arize AX

Production observability platform using OpenInference instrumentation.

```bash
pip install openinference-instrumentation-google-adk google-adk arize-otel
```

**Features:**
- Trace agent execution flows
- Performance evaluation with custom evaluators
- Real-time monitoring dashboards and alerts

---

### Freeplay

End-to-end platform for building and optimizing AI agents.

```bash
pip install freeplay-python-adk
```

```python
from freeplay_python_adk.client import FreeplayADK
from freeplay_python_adk.freeplay_observability_plugin import FreeplayObservabilityPlugin
from google.adk.runners import App

# Set environment variables:
# FREEPLAY_PROJECT_ID, FREEPLAY_API_KEY, FREEPLAY_API_URL

FreeplayADK.initialize_observability()

app = App(
    name="app",
    root_agent=root_agent,
    plugins=[FreeplayObservabilityPlugin()],
)
```

**Features:**
- Focused observability on agents, LLM calls, and tool calls
- Prompt management and versioning
- Batch testing for model/prompt variations
- Dataset management from production logs

---

### MLflow

OpenTelemetry span ingestion to MLflow Tracking Server.

```bash
pip install "mlflow>=3.6.0" google-adk opentelemetry-sdk opentelemetry-exporter-otlp-proto-http
```

**Start MLflow Server:**
```bash
mlflow server --backend-store-uri sqlite:///mlflow.db --port 5000
```

**Configure OpenTelemetry (before importing ADK):**
```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import SimpleSpanProcessor

exporter = OTLPSpanExporter(
    endpoint="http://localhost:5000/v1/traces",
    headers={"x-mlflow-experiment-id": "123"}
)

provider = TracerProvider()
provider.add_span_processor(SimpleSpanProcessor(exporter))
trace.set_tracer_provider(provider)

# Now import ADK components
from google.adk.agents import LlmAgent
```

> **Important**: Set tracer provider BEFORE importing ADK components.

---

### Monocle

Open-source observability with automatic instrumentation.

```bash
pip install monocle_apptrace google-adk
```

```python
from monocle_apptrace import setup_monocle_telemetry

setup_monocle_telemetry(workflow_name="my-adk-app")
```

**Exporter Configuration:**
```bash
# Console output (debugging)
export MONOCLE_EXPORTER="console"

# Local files (default)
export MONOCLE_EXPORTER="file"
```

Traces save to `./monocle/monocle_trace_{workflow_name}_{trace_id}_{timestamp}.json`

**Visualization:** Use Okahu Trace Visualizer VS Code extension for interactive analysis.

---

### Phoenix

Open-source, self-hosted observability platform.

```bash
pip install openinference-instrumentation-google-adk google-adk arize-phoenix-otel
```

```python
import os
from phoenix.otel import register

os.environ["PHOENIX_API_KEY"] = "YOUR_KEY"
os.environ["PHOENIX_COLLECTOR_ENDPOINT"] = "https://app.phoenix.arize.com/s/[your-space-name]"

tracer_provider = register(
    project_name="my-llm-app",
    auto_instrument=True
)
```

**Features:**
- Full trace capture (agent runs, tool calls, model requests)
- Custom and pre-built evaluators
- Self-hosted option for data control

---

### W&B Weave

Weights & Biases observability through OpenTelemetry traces.

```bash
pip install google-adk opentelemetry-sdk opentelemetry-exporter-otlp-proto-http
```

```python
import base64
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk import trace as trace_sdk
from opentelemetry.sdk.trace.export import SimpleSpanProcessor

# Configure exporter (set BEFORE importing ADK)
wandb_api_key = os.environ["WANDB_API_KEY"]
auth = base64.b64encode(f"api:{wandb_api_key}".encode()).decode()

exporter = OTLPSpanExporter(
    endpoint="https://trace.wandb.ai/otel/v1/traces",
    headers={"Authorization": f"Basic {auth}"}
)

tracer_provider = trace_sdk.TracerProvider()
tracer_provider.add_span_processor(SimpleSpanProcessor(exporter))
trace.set_tracer_provider(tracer_provider)

# Now import ADK components
from google.adk.agents import LlmAgent
```

> **Important**: Set tracer provider BEFORE importing ADK components.

---

## Comparison Table

| Tool | Type | Setup Effort | Key Strength |
|------|------|--------------|--------------|
| **Cloud Trace** | Google Cloud | Low | Native GCP integration |
| **BigQuery Analytics** | Google Cloud | Medium | SQL-based analysis, multimodal |
| **AgentOps** | SaaS | Very Low | 2-line setup, session replays |
| **Arize AX** | SaaS | Low | Production monitoring, alerts |
| **Freeplay** | SaaS | Low | Prompt management, batch testing |
| **MLflow** | Self-hosted | Medium | Experiment tracking integration |
| **Monocle** | Open-source | Low | Local file export, VS Code viz |
| **Phoenix** | Open-source/SaaS | Low | Self-hosted option, evaluators |
| **W&B Weave** | SaaS | Medium | W&B ecosystem integration |
