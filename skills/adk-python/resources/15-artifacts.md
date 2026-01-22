# Artifacts

> **Official Docs**: [Artifacts](https://google.github.io/adk-docs/artifacts/)

## Overview

Artifacts are **named, versioned binary data** managed by agents and tools. Unlike session state (for small text/JSON), artifacts handle files, images, audio, PDFs, and other binary formats.

| Feature | Session State | Artifacts |
|---------|---------------|-----------|
| Data Type | Text, JSON | Binary (files, images, audio) |
| Size | Small | Large |
| Versioning | No | Yes (auto-incremented) |
| Scope | Session | Session or User |

**Why Use Artifacts**:
- Handle non-textual data (PDFs, images, audio)
- Persist large data without cluttering session state
- Enable file uploads/downloads
- Share binary outputs across sessions
- Cache computationally expensive results

---

## Artifact Methods

### save_artifact()

Persists binary data as a versioned artifact.

```python
version = await context.save_artifact(
    filename="report.pdf",
    artifact=types.Part.from_bytes(
        data=pdf_bytes,
        mime_type="application/pdf"
    )
)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `filename` | `str` | Unique identifier. Use `"user:filename"` for user-scoped |
| `artifact` | `types.Part` | Binary data with MIME type |
| **Returns** | `int` | Version number assigned |

### load_artifact()

Retrieves a specific or latest version of an artifact.

```python
artifact = await context.load_artifact(
    filename="report.pdf",
    version=None  # None = latest
)

if artifact and artifact.inline_data:
    pdf_bytes = artifact.inline_data.data
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `filename` | `str` | Artifact identifier |
| `version` | `int` (optional) | Specific version; `None` for latest |
| **Returns** | `types.Part` | Artifact data or `None` if not found |

### list_artifacts()

Retrieves all artifact filenames in current scope.

```python
filenames = await context.list_artifacts()
# Returns: ["report.pdf", "chart.png", "user:profile.json"]
```

---

## ArtifactService

The service layer that handles artifact storage. Configure when creating the Runner.

### InMemoryArtifactService

In-memory storage for development and testing.

```python
from google.adk.artifacts import InMemoryArtifactService
from google.adk.runners import Runner

artifact_service = InMemoryArtifactService()

runner = Runner(
    agent=root_agent,
    app_name="my_app",
    artifact_service=artifact_service,
)
```

> **Warning**: All artifacts are **lost** when the process terminates.

### GcsArtifactService

Persistent storage using Google Cloud Storage for production.

```python
from google.adk.artifacts import GcsArtifactService

artifact_service = GcsArtifactService(bucket_name="my-bucket")

runner = Runner(
    agent=root_agent,
    app_name="my_app",
    artifact_service=artifact_service,
)
```

---

## Use Cases

### 1. Document Processing

Store and retrieve uploaded documents for analysis:

```python
from google.genai import types

async def process_document(tool_context: ToolContext, file_bytes: bytes, filename: str):
    """Save uploaded document for later processing."""
    artifact = types.Part.from_bytes(
        data=file_bytes,
        mime_type="application/pdf"
    )
    version = await tool_context.save_artifact(
        filename=filename,
        artifact=artifact
    )
    return f"Saved {filename} (version {version})"


async def retrieve_document(tool_context: ToolContext, filename: str):
    """Retrieve a saved document."""
    artifact = await tool_context.load_artifact(filename=filename)
    if artifact and artifact.inline_data:
        return artifact.inline_data.data
    return None
```

### 2. Report Generation

Generate and store reports as PDFs:

```python
async def generate_report(tool_context: ToolContext, data: dict):
    """Generate PDF report and save as artifact."""
    # Generate PDF bytes (using reportlab, weasyprint, etc.)
    pdf_bytes = create_pdf_report(data)

    report_artifact = types.Part.from_bytes(
        data=pdf_bytes,
        mime_type="application/pdf"
    )

    version = await tool_context.save_artifact(
        filename="monthly_report.pdf",
        artifact=report_artifact
    )

    return f"Report generated (version {version})"
```

### 3. Image Generation

Store generated charts or images:

```python
async def save_chart(tool_context: ToolContext, chart_bytes: bytes, name: str):
    """Save generated chart as artifact."""
    chart_artifact = types.Part.from_bytes(
        data=chart_bytes,
        mime_type="image/png"
    )

    await tool_context.save_artifact(
        filename=f"charts/{name}.png",
        artifact=chart_artifact
    )
```

### 4. User-Scoped Artifacts

Store user-specific data that persists across sessions:

```python
async def save_user_preferences(tool_context: ToolContext, prefs: dict):
    """Save user preferences (persists across sessions)."""
    import json

    prefs_artifact = types.Part.from_bytes(
        data=json.dumps(prefs).encode(),
        mime_type="application/json"
    )

    # "user:" prefix makes it user-scoped
    await tool_context.save_artifact(
        filename="user:preferences.json",
        artifact=prefs_artifact
    )
```

### 5. List and Manage Artifacts

```python
async def list_user_files(tool_context: ToolContext):
    """List all available artifacts."""
    filenames = await tool_context.list_artifacts()

    if not filenames:
        return "No saved files"

    return f"Available files: {', '.join(filenames)}"
```

---

## Complete Example

```python
# agent.py
from google.adk.agents import Agent
from google.adk.tools import ToolContext
from google.genai import types

async def save_note(tool_context: ToolContext, title: str, content: str) -> str:
    """Save a note as an artifact.

    Args:
        title: Note title (used as filename)
        content: Note content

    Returns:
        Confirmation message with version number
    """
    note_artifact = types.Part.from_bytes(
        data=content.encode("utf-8"),
        mime_type="text/plain"
    )

    version = await tool_context.save_artifact(
        filename=f"notes/{title}.txt",
        artifact=note_artifact
    )

    return f"Note '{title}' saved (version {version})"


async def read_note(tool_context: ToolContext, title: str) -> str:
    """Read a saved note.

    Args:
        title: Note title to retrieve

    Returns:
        Note content or error message
    """
    artifact = await tool_context.load_artifact(filename=f"notes/{title}.txt")

    if artifact and artifact.inline_data:
        return artifact.inline_data.data.decode("utf-8")

    return f"Note '{title}' not found"


async def list_notes(tool_context: ToolContext) -> str:
    """List all saved notes.

    Returns:
        List of note titles
    """
    filenames = await tool_context.list_artifacts()
    notes = [f for f in filenames if f.startswith("notes/")]

    if not notes:
        return "No notes saved"

    titles = [n.replace("notes/", "").replace(".txt", "") for n in notes]
    return f"Saved notes: {', '.join(titles)}"


root_agent = Agent(
    name="note_agent",
    model="gemini-2.5-flash",
    instruction="You are a note-taking assistant. Help users save, read, and manage their notes.",
    tools=[save_note, read_note, list_notes],
)
```

---

## Artifact Scopes

| Scope | Prefix | Description |
|-------|--------|-------------|
| Session | (none) | Available only in current session |
| User | `user:` | Persists across all user sessions |

```python
# Session-scoped (default)
await context.save_artifact("temp_file.pdf", artifact)

# User-scoped (persists across sessions)
await context.save_artifact("user:profile.json", artifact)
```

---

## Best Practices

1. **Use meaningful filenames**: Include paths for organization (`reports/2024/q1.pdf`)
2. **Choose appropriate scope**: Use `user:` prefix for data that should persist
3. **Handle missing artifacts**: Always check if `load_artifact()` returns `None`
4. **Set correct MIME types**: Helps with content negotiation and display
5. **Use GcsArtifactService in production**: InMemoryArtifactService loses data on restart
