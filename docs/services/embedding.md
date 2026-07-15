# Embedding Service

## Purpose

The embedding service provides a reusable ChaBo microservice for converting text
queries and documents into embedding vectors. It is intended for RAG indexing,
retrieval queries, and chatbot implementations that need a provider-independent
embedding API.

Issue: [#23](https://github.com/ChaBo-Project/chabo-microservices/issues/23)

## Architecture

The service is a standalone FastAPI application in `services/embedding-service`.
It wraps LangChain embedding integrations behind a small HTTP API:

```text
Client or orchestrator
  -> Embedding service
    -> LangChain Embeddings interface
      -> OpenAI, Google Gemini, or mock provider
```

The mock provider is deterministic and exists only for local development,
automated tests, and demos without cloud credentials.

## API

### Health

```bash
curl http://localhost:8085/health
```

Response:

```json
{
  "status": "healthy",
  "provider": "mock"
}
```

### Provider Metadata

```bash
curl http://localhost:8085/v1/embeddings/provider
```

### HuggingFace/TEI-Compatible Embeddings

Use `/embed` when a client expects a plain list of vectors:

```bash
curl http://localhost:8085/embed \
  -H "Content-Type: application/json" \
  -d '{"inputs":"What is ChaBo?","input_type":"query"}'
```

Response:

```json
[
  [0.12, -0.02, 0.42]
]
```

The request accepts either:

```json
{"inputs": "single text"}
```

or:

```json
{"inputs": ["first text", "second text"]}
```

### OpenAI-Compatible Embeddings

Use `/v1/embeddings` when a client expects OpenAI-style response metadata:

```bash
curl http://localhost:8085/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"input":["first text","second text"]}'
```

Response:

```json
{
  "object": "list",
  "data": [
    {
      "object": "embedding",
      "embedding": [0.12, -0.02, 0.42],
      "index": 0
    }
  ],
  "model": "mock-embedding",
  "provider": "mock",
  "dimensions": 8,
  "usage": {
    "prompt_tokens": 2,
    "total_tokens": 2
  }
}
```

## Configuration

| Variable | Default | Description |
| --- | --- | --- |
| `EMBEDDING_PROVIDER` | `mock` | `mock`, `openai`, `google`, or `gemini`. |
| `EMBEDDING_MODEL` | Provider default | Model name passed to LangChain. |
| `EMBEDDING_DIMENSIONS` | unset | Optional output dimension. Maps to OpenAI `dimensions` or Gemini `output_dimensionality`. |
| `EMBEDDING_MOCK_DIMENSIONS` | `8` | Vector size used by the mock provider. |
| `EMBEDDING_MAX_BATCH_SIZE` | `100` | Maximum number of texts per request. |
| `EMBEDDING_API_KEY` | unset | Optional generic provider API key override. |
| `EMBEDDING_BASE_URL` | unset | Optional OpenAI-compatible base URL for proxies or gateways. |
| `EMBEDDING_TASK_TYPE` | unset | Optional Gemini task type, such as `RETRIEVAL_QUERY` or `RETRIEVAL_DOCUMENT`. |
| `OPENAI_API_KEY` | unset | Required when `EMBEDDING_PROVIDER=openai`, unless `EMBEDDING_API_KEY` is set. |
| `GOOGLE_API_KEY` | unset | Required when `EMBEDDING_PROVIDER=google`, unless `EMBEDDING_API_KEY` or `GEMINI_API_KEY` is set. |
| `GEMINI_API_KEY` | unset | Alias accepted for Gemini credentials. |

Provider defaults:

| Provider | Default model |
| --- | --- |
| `mock` | `mock-embedding` |
| `openai` | `text-embedding-3-small` |
| `google` / `gemini` | `gemini-embedding-2-preview` |

## Deployment

### Docker

```bash
docker build -f services/embedding-service/Dockerfile -t chabo-embedding-service .
docker run --rm -p 8085:8080 \
  -e EMBEDDING_PROVIDER=mock \
  chabo-embedding-service
```

### Docker Compose

Start the standalone service through the `ai-services` profile:

```bash
COMPOSE_PROFILES=ai-services docker compose \
  -f docker-compose/docker-compose.yml \
  up --build embedding-service
```

OpenAI example:

```bash
EMBEDDING_PROVIDER=openai
OPENAI_API_KEY=...
EMBEDDING_MODEL=text-embedding-3-small
```

Google Gemini example:

```bash
EMBEDDING_PROVIDER=google
GOOGLE_API_KEY=...
EMBEDDING_MODEL=gemini-embedding-2-preview
EMBEDDING_DIMENSIONS=768
```

## Security

Do not commit provider API keys. Configure them through local `.env` files,
GitHub Actions secrets, or server-side secret storage.

The service does not add API authentication by itself. Put it behind private
networking, an API gateway, or reverse-proxy authentication before exposing it
outside trusted infrastructure.

## Operations

Use `/health` for liveness checks. For provider debugging, check:

- `EMBEDDING_PROVIDER`
- `EMBEDDING_MODEL`
- provider API key environment variables
- request batch size
- vector dimensions expected by the downstream vector database

## References

- [LangChain OpenAIEmbeddings](https://docs.langchain.com/oss/python/integrations/embeddings/openai)
- [LangChain GoogleGenerativeAIEmbeddings](https://docs.langchain.com/oss/python/integrations/embeddings/google_generative_ai)
