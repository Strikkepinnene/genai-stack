# GenAI Stack Copilot Instructions

## Architecture Overview

This is a multi-service RAG (Retrieval-Augmented Generation) application with Neo4j knowledge graph storage and multiple UI frontends. The system demonstrates LLM integration patterns with both pure generation and knowledge-enhanced responses.

### Core Components

- **Neo4j Database**: Central knowledge graph storing Stack Overflow Q&A data with vector embeddings
- **LLM Providers**: Supports Ollama (local), OpenAI GPT, Claude, and AWS Bedrock models
- **Embedding Models**: Configurable via `EMBEDDING_MODEL` env var (sentence_transformer, openai, ollama, aws)
- **Multiple Apps**: 5 containerized applications sharing common chain logic

## Key Files & Patterns

### Chain Architecture (`chains.py`)
- **Dual Mode Pattern**: All apps support both RAG-enabled and pure LLM responses
- **Chain Functions**: `configure_llm_only_chain()` vs `configure_qa_rag_chain()`
- **Model Loading**: Use `load_llm()` and `load_embedding_model()` with config dict pattern
- **Provider Detection**: Model names trigger different providers (e.g., "gpt-4" → OpenAI, "llama2" → Ollama)

### Environment Configuration
Always use environment variables from `.env` for:
- `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD` (database connection)
- `OLLAMA_BASE_URL` (for local Ollama instance)
- `LLM` and `EMBEDDING_MODEL` (model selection)
- API keys for external providers (OpenAI, AWS, Google)

### Service Communication Patterns
- **FastAPI Streaming**: `api.py` uses `QueueCallback` + `EventSourceResponse` for SSE streaming
- **Streamlit Integration**: `bot.py` uses `StreamHandler` for real-time token display
- **CORS Setup**: Frontend at `:8505` connects to API at `:8504` with open CORS policy

## Development Workflows

### Docker Compose Commands
```bash
# Start all services
docker compose up

# Rebuild after changes
docker compose up --build

# Auto-rebuild on file changes (development)
docker compose watch

# Platform-specific profiles
docker compose --profile linux up        # Linux with Ollama container
docker compose --profile linux-gpu up    # Linux with GPU support
```

### Service Architecture
- **bot** (`:8501`): Streamlit UI with RAG toggle
- **loader** (`:8502`): Stack Overflow data import tool  
- **pdf_bot** (`:8503`): PDF document Q&A interface
- **api** (`:8504`): FastAPI backend with streaming endpoints
- **front-end** (`:8505`): Svelte SPA consuming the API
- **database** (`:7474/:7687`): Neo4j with APOC plugins

## Code Patterns

### Vector Index Management
Always call `create_vector_index(neo4j_graph)` after Neo4j connection:
```python
# Creates indexes: stackoverflow (Question.embedding), top_answers (Answer.embedding)
create_vector_index(neo4j_graph)
```

### Callback Patterns
- **Streaming**: Implement `BaseCallbackHandler.on_llm_new_token()` for real-time responses
- **Queue-based**: Use `QueueCallback` pattern for async streaming (see `api.py`)
- **UI Updates**: Use `StreamHandler` for Streamlit containers (see `bot.py`)

### Frontend Integration
- **SSE Consumption**: Frontend uses `EventSource` to consume `/query-stream` endpoint
- **State Management**: Svelte stores in `chat.store.js` and `generation.store.js`
- **RAG Toggle**: UI controls switch between pure LLM and RAG-enhanced responses

## Critical Implementation Notes

- **Environment Mapping**: Set `os.environ["NEO4J_URL"] = url` for LangChain Neo4j compatibility
- **Model Dimensions**: Embedding dimensions vary by provider (sentence_transformer: 384, openai: 1536, ollama: 4096)
- **Health Checks**: Database must be healthy before dependent services start
- **Watch Mode**: Each service has specific ignore patterns to prevent unnecessary rebuilds
- **Ticket Generation**: Uses graph queries to find high-quality examples for support ticket templates

## Testing & Debugging

- **Neo4j Browser**: Access at `http://localhost:7474` to inspect graph data
- **API Health**: Check `http://localhost:8504/` for API status
- **Log Streaming**: Use `BaseLogger()` class for consistent logging across services
- **Development Ports**: Each service exposes different ports for isolated testing

When modifying chains, always test both RAG modes. When adding new models, update the provider detection logic in `load_llm()` and ensure proper streaming callback support.