# Deployment

> **Reference**: https://google.github.io/adk-docs/deploy/

## Overview

ADK supports multiple deployment options for production environments.

| Platform | Description | Best For |
|----------|-------------|----------|
| **Agent Engine** | Fully managed auto-scaling service on Vertex AI | Production, managed infrastructure |
| **Cloud Run** | Serverless container platform | Serverless, container-based apps |
| **GKE** | Managed Kubernetes service | Fine-grained control, open models |
| **Custom** | Any container infrastructure | Offline/disconnected environments |

---

## Agent Engine

> **Reference**: https://google.github.io/adk-docs/deploy/agent-engine/

Vertex AI Agent Engine is Google Cloud's fully managed solution for deploying AI agents.

### Key Features

- Automatic scaling
- Managed infrastructure (no container management)
- End-to-end support for ADK agents
- Built-in session management

### Deployment Paths

| Path | Description | Use Case |
|------|-------------|----------|
| **Standard** | Manual setup with ADK CLI | Production, existing GCP projects |
| **Agent Starter Pack (ASP)** | Accelerated setup with templates | Development, testing |

---

### Standard Deployment

> **Reference**: https://google.github.io/adk-docs/deploy/agent-engine/deploy/

#### Prerequisites

1. **Google Cloud Project** with:
   - Active billing account
   - Vertex AI API enabled
   - Cloud Resource Manager API enabled

2. **Local Authentication**:
```bash
gcloud auth login
gcloud auth application-default login
gcloud config set project $PROJECT_ID
```

3. **Project Structure**:
```
agent_folder/
├── .env
├── __init__.py
└── agent.py          # Must have root_agent variable
```

#### Deploy Command

```bash
adk deploy agent_engine \
    --project=$PROJECT_ID \
    --region=$LOCATION_ID \
    --display_name="My Agent" \
    agent_folder
```

#### Post-Deployment Output

Successful deployment provides:
- **Resource Name**: Full path identifying your deployed agent
- **Resource ID**: Numeric identifier for API calls
- **Logs URL**: Deployment diagnostics

#### Accessing Deployed Agent

REST API URL structure:
```
https://$LOCATION_ID-aiplatform.googleapis.com/v1/projects/$PROJECT_ID/locations/$LOCATION_ID/reasoningEngines/$RESOURCE_ID:query
```

---

### Agent Starter Pack (ASP)

> **Reference**: https://google.github.io/adk-docs/deploy/agent-engine/asp/

Accelerated deployment tool for development and testing (not production).

#### Prerequisites

- Google Cloud account with admin access
- Empty project with billing enabled
- Python environment with `uv`
- Google Cloud CLI (`gcloud`)
- Make tool

#### Setup Process

```bash
# 1. Enhance ADK project with deployment files
uvx agent-starter-pack enhance --adk -d agent_engine

# 2. Authenticate
gcloud auth application-default login
gcloud config set project $PROJECT_ID

# 3. Deploy
make backend
```

#### Project Structure After ASP

```
project/
├── app/                 # Core application logic
├── .cloudbuild/         # CI/CD configurations
├── deployment/          # Infrastructure scripts
├── tests/               # Testing suites
├── Makefile
└── pyproject.toml
```

**Note**: ASP configures additional GCP resources beyond strict requirements. Review before production use.

---

### Testing Deployed Agents

> **Reference**: https://google.github.io/adk-docs/deploy/agent-engine/test/

#### View in Console

Navigate to: `https://console.cloud.google.com/vertex-ai/agents/agent-engines`

#### Python SDK Testing

```python
from google.adk.sessions import VertexAiSessionService

# Create remote app instance
remote_app = VertexAiSessionService(
    project=PROJECT_ID,
    location=LOCATION_ID,
    reasoning_engine_id=RESOURCE_ID,
)

# Create session
remote_session = await remote_app.async_create_session(user_id="user_123")

# Query agent
async for event in remote_app.async_stream_query(
    user_id="user_123",
    session_id=remote_session.id,
    message="Hello!"
):
    print(event)
```

#### Cleanup

```python
# Delete deployed instance to avoid charges
await remote_app.delete(force=True)
```

---

## Cloud Run

> **Reference**: https://google.github.io/adk-docs/deploy/cloud-run/

Serverless container platform for ADK agents.

### Using ADK CLI (Recommended)

```bash
# Minimal deployment
adk deploy cloud_run \
    --project=$PROJECT_ID \
    --region=$LOCATION_ID \
    agent_folder

# With options
adk deploy cloud_run \
    --project=$PROJECT_ID \
    --region=$LOCATION_ID \
    --service_name="my-agent-service" \
    --with_ui \
    --port=8080 \
    agent_folder
```

#### CLI Options

| Option | Description |
|--------|-------------|
| `--service_name` | Cloud Run service name |
| `--app_name` | ADK API server application name |
| `--with_ui` | Deploy development UI |
| `--port` | Container port (default: 8000) |
| `--agent_engine_id` | Vertex AI Agent Engine integration |

### Using gcloud CLI

#### 1. Create main.py

```python
from google.adk.cli.fast_api import get_fast_api_app

app = get_fast_api_app(
    agents_dir=".",
    session_service_uri="sqlite+aiosqlite:///./sessions.db",
    web=False,
)
```

#### 2. Create Dockerfile

```dockerfile
FROM python:3.13-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Create non-root user
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8080

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

#### 3. Deploy

```bash
gcloud run deploy my-agent \
    --source . \
    --region=$LOCATION_ID \
    --allow-unauthenticated
```

### Environment Variables

```bash
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=us-central1
GOOGLE_GENAI_USE_VERTEXAI=True  # or False with GOOGLE_API_KEY
```

### Testing

```bash
# Get service URL
SERVICE_URL=$(gcloud run services describe my-agent \
    --region=$LOCATION_ID \
    --format='value(status.url)')

# Create session
curl -X POST "$SERVICE_URL/apps/agent_folder/users/user_123/sessions/session_456" \
    -H "Authorization: Bearer $(gcloud auth print-identity-token)"

# Query agent (SSE)
curl -X POST "$SERVICE_URL/run_sse" \
    -H "Authorization: Bearer $(gcloud auth print-identity-token)" \
    -H "Content-Type: application/json" \
    -d '{
        "appName": "agent_folder",
        "userId": "user_123",
        "sessionId": "session_456",
        "newMessage": {"role": "user", "parts": [{"text": "Hello!"}]}
    }'
```

---

## GKE

> **Reference**: https://google.github.io/adk-docs/deploy/gke/

Google Kubernetes Engine deployment for fine-grained control.

### Using ADK CLI (Automated)

```bash
adk deploy gke \
    --project=$PROJECT_ID \
    --region=$LOCATION_ID \
    --cluster=$CLUSTER_NAME \
    agent_folder
```

This command handles:
- Container image build
- Push to Artifact Registry
- Manifest generation
- Cluster deployment

### Manual Deployment

#### 1. Create GKE Cluster

```bash
gcloud container clusters create adk-cluster \
    --region=$LOCATION_ID \
    --workload-pool=$PROJECT_ID.svc.id.goog \
    --num-nodes=1
```

#### 2. Create Dockerfile

```dockerfile
FROM python:3.13-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 8080

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

#### 3. Build and Push Image

```bash
gcloud builds submit --tag gcr.io/$PROJECT_ID/adk-agent:latest
```

#### 4. Create Kubernetes Manifests

**deployment.yaml**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adk-agent
spec:
  replicas: 1
  selector:
    matchLabels:
      app: adk-agent
  template:
    metadata:
      labels:
        app: adk-agent
    spec:
      serviceAccountName: adk-agent-sa
      containers:
      - name: agent
        image: gcr.io/PROJECT_ID/adk-agent:latest
        ports:
        - containerPort: 8080
        env:
        - name: GOOGLE_CLOUD_PROJECT
          value: "PROJECT_ID"
        - name: GOOGLE_CLOUD_LOCATION
          value: "us-central1"
        - name: GOOGLE_GENAI_USE_VERTEXAI
          value: "True"
        resources:
          requests:
            memory: "128Mi"
            cpu: "500m"
```

**service.yaml**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: adk-agent-service
spec:
  type: LoadBalancer
  selector:
    app: adk-agent
  ports:
  - port: 80
    targetPort: 8080
```

#### 5. Configure Workload Identity

```bash
# Create Kubernetes service account
kubectl create serviceaccount adk-agent-sa

# Create IAM binding
gcloud iam service-accounts add-iam-policy-binding \
    adk-agent-sa@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[default/adk-agent-sa]"

# Annotate Kubernetes service account
kubectl annotate serviceaccount adk-agent-sa \
    iam.gke.io/gcp-service-account=adk-agent-sa@$PROJECT_ID.iam.gserviceaccount.com

# Grant Vertex AI User role
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:adk-agent-sa@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/aiplatform.user"
```

#### 6. Deploy

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Get external IP
kubectl get service adk-agent-service
```

### Common Issues

| Issue | Solution |
|-------|----------|
| Permission denied (403) | Verify IAM bindings for Vertex AI User role |
| Read-only database | Exclude SQLite files from container image |
| Missing Workload Identity | Ensure proper service account setup |

---

## Deployment Comparison

| Feature | Agent Engine | Cloud Run | GKE |
|---------|--------------|-----------|-----|
| Management | Fully managed | Serverless | Self-managed |
| Scaling | Auto | Auto | Configurable |
| Container control | None | Limited | Full |
| Open models | No | Yes | Yes |
| Cost model | Pay-per-use | Pay-per-request | Pay-per-node |
| Setup complexity | Low | Low | High |

---

## Production Checklist

### Security
- [ ] API keys in Secret Manager or environment variables
- [ ] CORS origins explicitly configured
- [ ] HTTPS enabled
- [ ] Authentication/authorization layer

### Session Management
- [ ] DatabaseSessionService for persistence
- [ ] Session cleanup policy
- [ ] Backup strategy

### Monitoring
- [ ] Logging level configured (INFO for production)
- [ ] Cloud Trace enabled (GCP)
- [ ] Error alerting setup

### Performance
- [ ] `max_llm_calls` configured (prevent infinite loops)
- [ ] Appropriate resource allocation
- [ ] Load balancing (multi-instance)

---

## Official Documentation Links

| Topic | URL |
|-------|-----|
| Deploy Overview | https://google.github.io/adk-docs/deploy/ |
| Agent Engine | https://google.github.io/adk-docs/deploy/agent-engine/ |
| Standard Deployment | https://google.github.io/adk-docs/deploy/agent-engine/deploy/ |
| Agent Starter Pack | https://google.github.io/adk-docs/deploy/agent-engine/asp/ |
| Cloud Run | https://google.github.io/adk-docs/deploy/cloud-run/ |
| GKE | https://google.github.io/adk-docs/deploy/gke/ |
