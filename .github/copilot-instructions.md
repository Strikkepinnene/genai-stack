# GenAI Stack - AI Coding Agent Instructions

## Architecture Overview

This is a **multi-service RAG (Retrieval-Augmented Generation)** application built with LangChain, Neo4j, and multiple LLM providers. Five containerized services (`bot`, `loader`, `pdf_bot`, `api`, `front-end`) share common chain logic defined in `chains.py`.

### Core Components
- **chains.py**: Defines embedding models, LLMs, and two chain types:
  - `configure_llm_only_chain()`: Pure LLM responses
  - `configure_qa_rag_chain()`: Vector + Knowledge Graph enhanced responses
- **Neo4j Graph DB**: Stores Stack Overflow Q&A with vector embeddings for similarity search
- **api.py**: FastAPI backend with SSE streaming endpoints
- **bot.py**: Streamlit chat interface with ticket generation
- **loader.py**: Imports Stack Overflow data and creates embeddings
- **pdf_bot.py**: PDF document Q&A with vector storage

## Critical Patterns

### 1. Dual Chain Pattern
Every service implements both LLM-only and RAG modes. Always provide both options:
```python
llm_chain = configure_llm_only_chain(llm)
rag_chain = configure_qa_rag_chain(llm, embeddings, ...)
```

### 2. Provider Abstraction
`load_llm()` and `load_embedding_model()` detect providers by name patterns:
- LLM: "gpt-4" → OpenAI, "claudev2" → AWS Bedrock, else → Ollama
- Embeddings: "openai" → OpenAIEmbeddings (1536D), "ollama" → OllamaEmbeddings (4096D), else → HuggingFace (384D)

### 3. Environment Variable Remapping
**Always** set `os.environ["NEO4J_URL"] = url` after loading `NEO4J_URI` from `.env` for LangChain Neo4j compatibility.

### 4. Vector Index Creation
**Always** call `create_vector_index(neo4j_graph)` after Neo4j connection. Creates two indices:
- `stackoverflow` index on `Question.embedding`
- `top_answers` index on `Answer.embedding`

### 5. Streaming Implementations
Two different patterns for real-time responses:
- **FastAPI (api.py)**: `QueueCallback` + `EventSourceResponse` for SSE
- **Streamlit (bot.py, pdf_bot.py)**: `StreamHandler` updates container on each token

### 6. Custom Retrieval Query
The RAG chain uses a custom Cypher query in `configure_qa_rag_chain()` that:
- Finds similar questions via vector search
- Retrieves top 2 answers ordered by `is_accepted` and `score`
- Returns formatted context with question + answers + source links

## Developer Workflows

### Docker Profiles
Platform-specific Ollama configurations:
- **MacOS/Windows**: Run Ollama locally, use `OLLAMA_BASE_URL=http://host.docker.internal:11434`
- **Linux**: Use `docker compose --profile linux up` with `OLLAMA_BASE_URL=http://llm:11434`
- **Linux GPU**: Use `docker compose --profile linux-gpu up` with `OLLAMA_BASE_URL=http://llm-gpu:11434`

### Watch Mode
Enable auto-rebuild on file changes: `docker compose watch`
- Each service has ignore patterns in `x-develop.watch` to avoid cross-service rebuilds
- Front-end uses `sync` action for instant hot-reload

### Health Checks
Services depend on health checks:
- Database must be healthy before app services start
- API must be healthy before front-end starts
- Use `wget localhost:7474` for Neo4j, `wget localhost:8504/` for API

## Common Tasks

### Adding a New LLM Provider
1. Update `load_llm()` in `chains.py` with provider detection logic
2. Import the LangChain provider class (e.g., `from langchain_x import ChatX`)
3. Return with `streaming=True` configured

### Adding New Embedding Provider
1. Update `load_embedding_model()` in `chains.py`
2. Specify the correct dimension (384, 768, 1536, or 4096)
3. Test with `loader.py` to ensure embeddings are created

### Modifying RAG Behavior
- Edit the Cypher `retrieval_query` in `configure_qa_rag_chain()` to change how context is retrieved
- Adjust `search_kwargs={"k": 2}` to return more/fewer results
- Modify prompt templates in `general_system_template`

### Creating New Services
Follow the existing pattern:
1. Create `service.py` with Neo4j connection and `create_vector_index()` call
2. Load embeddings and LLM via `load_embedding_model()` and `load_llm()`
3. Configure both chains (`llm_chain` and `rag_chain`)
4. Add Dockerfile and service definition in `docker-compose.yml`
5. Add ignore patterns in other services' `x-develop.watch` sections

## Key Files Reference
- **chains.py**: All chain logic, model loading, ticket generation
- **utils.py**: Vector index creation, constraints, doc formatting
- **docker-compose.yml**: Service orchestration, profiles, watch mode
- **env.example**: All environment variables with descriptions
- **requirements.txt**: Python dependencies (note: Streamlit pinned to 1.32.1)

## Testing Strategy
The project currently has no automated tests. When developing:
1. Start services with `docker compose up`
2. Manually test each service via its web UI
3. Verify Neo4j data at http://localhost:7474
4. Test both RAG enabled/disabled modes
5. Check logs for errors: `docker compose logs <service>`
