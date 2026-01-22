# Grounding in ADK

> **Official Docs**:
> - [Grounding Overview](https://google.github.io/adk-docs/grounding/)
> - [Google Search Grounding](https://google.github.io/adk-docs/grounding/google_search_grounding/)
> - [Vertex AI Search Grounding](https://google.github.io/adk-docs/grounding/vertex_ai_search_grounding/)

## What is Grounding?

Grounding connects LLMs to external knowledge sources, enabling agents to access information beyond their training data cutoff. This is essential for:

- **Real-time information**: Weather, news, stock prices, current events
- **Enterprise data**: Internal documents, private knowledge bases
- **Accuracy**: Reducing hallucinations by anchoring responses to factual sources
- **Attribution**: Providing source citations for transparency

---

## Grounding Types Comparison

| Feature | Google Search Grounding | Vertex AI Search Grounding |
|---------|------------------------|---------------------------|
| **Data Source** | Public web | Private enterprise data |
| **Use Case** | Current events, general knowledge | Internal documents, company data |
| **Setup** | API key only | GCP project + datastore |
| **Privacy** | Public data | Data stays in GCP |
| **Tool** | `google_search` | `VertexAiSearchTool` |

---

## Google Search Grounding

Connects agents to real-time web information via Google Search.

### How It Works

```
User Query → LLM Analysis → google_search Tool → Google Search Index
                                    ↓
Response with Citations ← Context Injection ← Relevant Snippets
```

1. User submits a query
2. LLM determines if external information is needed
3. If yes, `google_search` tool queries Google's search index
4. Relevant snippets and web pages are retrieved
5. Content is injected into model context
6. LLM generates a grounded response with source attribution

### Configuration

```python
from google.adk.agents import Agent
from google.adk.tools import google_search

root_agent = Agent(
    name="search_agent",
    model="gemini-2.5-flash",
    instruction="""You are a helpful assistant with access to current information.
    Use Google Search when you need:
    - Real-time data (weather, news, prices)
    - Facts that may have changed since your training
    - Verification of current information

    Always cite your sources.""",
    tools=[google_search],
)
```

### Requirements

| Requirement | Description |
|-------------|-------------|
| Python | 3.10+ |
| ADK | `pip install google-adk` |
| API Key | `GOOGLE_API_KEY` environment variable |

### Response Metadata

Responses include `groundingMetadata`:

```python
{
    "groundingChunks": [
        {"web": {"title": "Source Title", "uri": "https://example.com"}}
    ],
    "groundingSupports": [
        {"segment": {"text": "Answer text"}, "groundingChunkIndices": [0]}
    ],
    "searchEntryPoint": {
        "renderedContent": "<html>...</html>"  # Search suggestion chips
    }
}
```

---

## Vertex AI Search Grounding

Connects agents to indexed enterprise documents and private data repositories.

### How It Works

```
User Query → LLM Analysis → VertexAiSearchTool → Vertex AI Search Datastore
                                    ↓
Response with Citations ← Context Injection ← Ranked Document Chunks
```

1. User submits a query
2. LLM determines if enterprise documents are needed
3. `VertexAiSearchTool` queries your Vertex AI Search datastore
4. Relevant document chunks are retrieved and ranked
5. Content is injected into model context
6. LLM generates a response grounded in enterprise data

### Configuration

```python
from google.adk.agents import Agent
from google.adk.tools import VertexAiSearchTool

# Datastore ID format: projects/{project}/locations/{location}/collections/default_collection/dataStores/{datastore_id}
DATASTORE_ID = "projects/my-project/locations/global/collections/default_collection/dataStores/my-datastore"

root_agent = Agent(
    name="enterprise_agent",
    model="gemini-2.5-flash",
    instruction="""You are an enterprise assistant with access to company documents.
    Use the search tool to find relevant information from internal knowledge bases.
    Always cite the source documents.""",
    tools=[VertexAiSearchTool(data_store_id=DATASTORE_ID)],
)
```

### Requirements

| Requirement | Description |
|-------------|-------------|
| GCP Project | Vertex AI API enabled |
| Datastore | Configured Vertex AI Search datastore with indexed documents |
| Authentication | gcloud CLI authentication |
| Environment | See environment variables below |

### Environment Variables

```env
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=us-central1
```

### Response Metadata

```python
{
    "groundingChunks": [
        {
            "retrievedContext": {
                "title": "Document Title",
                "uri": "gs://bucket/document.pdf",
                "id": "doc-123"
            }
        }
    ],
    "groundingSupports": [
        {"segment": {"text": "Answer text"}, "groundingChunkIndices": [0]}
    ],
    "retrievalQueries": ["actual search query executed"]
}
```

---

## Use Cases

### Google Search Grounding

| Use Case | Example Query |
|----------|---------------|
| Current events | "What are today's top news stories?" |
| Real-time data | "What's the weather in Seoul?" |
| Price information | "Current Bitcoin price" |
| Recent updates | "Latest Python 3.13 features" |
| Fact verification | "When was the last solar eclipse?" |

### Vertex AI Search Grounding

| Use Case | Example Query |
|----------|---------------|
| Policy lookup | "What is our vacation policy?" |
| Product documentation | "How do I configure feature X?" |
| Internal knowledge | "What were Q3 sales numbers?" |
| Compliance | "What are the data retention requirements?" |
| Onboarding | "What's the new employee setup process?" |

---

## Combined Grounding

You can use both grounding methods in a single agent:

```python
from google.adk.agents import Agent
from google.adk.tools import google_search, VertexAiSearchTool

DATASTORE_ID = "projects/my-project/locations/global/..."

root_agent = Agent(
    name="hybrid_agent",
    model="gemini-2.5-flash",
    instruction="""You are a helpful assistant with access to:
    1. Google Search - for current events and public information
    2. Company knowledge base - for internal policies and documents

    Choose the appropriate source based on the query type.
    Always cite your sources.""",
    tools=[
        google_search,
        VertexAiSearchTool(data_store_id=DATASTORE_ID),
    ],
)
```

---

## Best Practices

1. **Clear instructions**: Tell the agent when to use grounding tools
2. **Source attribution**: Always instruct agents to cite sources
3. **Appropriate tool selection**: Use Google Search for public data, Vertex AI for private data
4. **Display metadata**: Use `groundingMetadata` to show citations in UI
5. **Search suggestions**: Render `searchEntryPoint` for related query exploration

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `google_search` not working | Verify `GOOGLE_API_KEY` is set |
| Vertex AI authentication error | Run `gcloud auth application-default login` |
| Datastore not found | Check datastore ID format and permissions |
| No grounding metadata | Ensure tool was actually invoked (check tool calls) |
| Outdated information | Verify Google Search is being used for real-time queries |
