# ChaBo Project Analysis, Recovery Runbook, and Current Architecture

This document captures what we found while analyzing the ChaBo project, the issues we hit while trying to run it, the fixes we implemented, the environment variables required, and the current working architecture.

## 1. Executive Summary

ChaBo is a Retrieval-Augmented Generation application built around FastAPI, LangServe, LangGraph, LangChain, Hugging Face services, and Qdrant.

The original project was structured to support:

- A FastAPI backend on port `7860`.
- LangServe routes for text chat and file-assisted chat.
- Embedding through Hugging Face endpoints.
- Vector search through Qdrant.
- Reranking through a Hugging Face endpoint.
- Generation through several possible LLM providers.
- Optional ChatUI frontend through Docker Compose.
- Optional local infrastructure through Docker Compose profiles.

The project did not run cleanly out of the box in this workspace because several configuration values were placeholders or pointed to remote services that were not available. We moved the running stack to:

- Local Qdrant.
- Local Hugging Face Text Embeddings Inference (TEI) for embeddings.
- Local TEI for reranking.
- Hugging Face Inference Providers Router for generation.
- RAG-Multi-Corpus as the loaded document corpus.

The current backend is healthy and reachable at:

```text
http://127.0.0.1:7860
```

API docs:

```text
http://127.0.0.1:7860/docs
```

The current Qdrant collection is:

```text
rag_multi_corpus
```

It currently contains `2115` points with `768` dimensional vectors using cosine distance.

## 2. First Analysis Insights

### 2.1 Repository Shape

Important files and directories:

```text
src/main.py
src/components/orchestration/
src/components/retriever/
src/components/generator/
src/components/ingestor/
docker-compose/docker-compose.yml
docker-compose/.env.example
params.cfg
tests/
data/rag_multi_corpus/
```

The main application entrypoint is `src/main.py`. It initializes:

- The retriever from `params.cfg`.
- The generator from `params.cfg` and environment variables.
- The LangGraph workflow.
- The FastAPI and LangServe routes.

The two main LangServe routes are:

```text
/chatfed-ui-stream
/chatfed-with-file-stream
```

The health endpoint is:

```text
/health
```

### 2.2 Original Runtime Design

The initial architecture was already modular:

```text
Client or ChatUI
  -> FastAPI / LangServe
  -> UI adapter
  -> LangGraph workflow
  -> optional file ingestion
  -> metadata filter extraction
  -> embedding
  -> Qdrant vector search
  -> reranking
  -> LLM generation
  -> streamed answer with citations and sources
```

The workflow is defined in `src/components/orchestration/workflow.py`:

```text
START
  -> ingest
  -> extract_filters
  -> retrieve
  -> generate
  -> END
```

### 2.3 Main Configuration Findings

The important configuration file is `params.cfg`.

At the start, the file contained placeholder values for:

- Hugging Face embedding endpoint.
- Hugging Face reranker endpoint.
- Qdrant URL and collection.
- Generator endpoint/model.
- Metadata filters from an older agriculture-oriented dataset.

The Docker Compose setup already had useful profiles:

- `infra`: starts local Qdrant.
- `local`: starts local TEI services for embedding and reranking.

That meant we did not need a remote Qdrant instance or paid Hugging Face dedicated embedding/reranker endpoints. We could run the retrieval side locally.

## 3. Challenges Faced While Running the System

### 3.1 Placeholder Remote Endpoints

The original `params.cfg` was prepared for remote Hugging Face endpoints, but the endpoint URLs were placeholders. Because there was no working hosted embedding endpoint, hosted reranker endpoint, or generator endpoint, the app could not complete the RAG flow.

Resolution:

- Replaced the embedding endpoint with local TEI:

```ini
embedding_endpoint_url = http://tei-embedding:80
```

- Replaced the reranker endpoint with local TEI:

```ini
reranker_endpoint_url = http://tei-reranker:80
```

- Replaced the Qdrant target with the local Docker Qdrant service:

```ini
url = http://qdrant:6333
port = 6333
collection = rag_multi_corpus
```

### 3.2 Generator Endpoint Was Missing

The generator originally used:

```ini
PROVIDER = huggingface
MODEL = https://xxxxxxxxxxx.huggingface.cloud
```

That expected a dedicated Hugging Face TGI endpoint. We did not have one.

Resolution:

- Added a new provider called `huggingface_router`.
- Used the Hugging Face Inference Providers Router as an OpenAI-compatible endpoint.
- Configured the generator to use:

```ini
PROVIDER = huggingface_router
MODEL = deepseek-ai/DeepSeek-V4-Flash:deepinfra
OPENAI_BASE_URL = https://router.huggingface.co/v1
```

We tested multiple router model/provider combinations with the existing Hugging Face token. The selected model returned HTTP `200` and produced a valid response.

### 3.3 Local TEI Model Startup and Health

The local TEI containers need to download and serve models. First startup can take time.

The initial reranker configuration was not reliable enough for the local CPU setup. We changed the reranker model to:

```text
cross-encoder/ms-marco-MiniLM-L-6-v2
```

We also increased the reranker healthcheck startup allowance:

```yaml
retries: 20
start_period: 120s
```

### 3.4 Embedding Model and Vector Size Alignment

The Qdrant collection must use the same vector size as the embedding model.

Current embedding model:

```text
BAAI/bge-base-en-v1.5
```

Current Qdrant vector size:

```text
768
```

This matches `BAAI/bge-base-en-v1.5`.

### 3.5 Reranker Returned Flat Scores

The local reranker sometimes returned identical scores, for example `0.5` for every candidate. That caused useful vector-search ordering to be scrambled.

Resolution:

- Added a guard in `src/components/retriever/retriever_orchestrator.py`.
- If all reranker scores are identical, the retriever raises internally and falls back to vector-search order.

This fixed the observed issue where relevant ZX Bank chunks were retrieved by vector search but then poorly reordered by flat reranker scores.

### 3.6 RAG-Multi-Corpus Folder and Format

The folder was named:

```text
../RAG-Multi-Corpus
```

not:

```text
RAG-MULTI-CORPUS
```

The corpus includes raw files in several formats, but the best ingestion path was the prepared semantic chunk JSON files:

```text
parsed-chunks-less-tokens/*_optimised_prompt_chunks.json
```

Resolution:

- Copied the prepared chunk JSON files into this repo under:

```text
data/rag_multi_corpus/parsed-chunks-less-tokens/
```

- Added a dedicated importer:

```text
src/components/ingestor/import_rag_multi_corpus.py
```

### 3.7 Docker and Localhost Access

Some Docker and localhost checks required elevated execution in the coding sandbox. The service itself was healthy, but sandboxed `curl` could not always reach localhost until run with the correct permission.

Resolution:

- Verified health through Docker and localhost checks after requesting runtime access.
- Confirmed:

```text
http://127.0.0.1:7860/health -> {"status":"healthy"}
```

## 4. Step-by-Step Solutions Implemented

### Step 1: Switched Retrieval Infrastructure to Local Services

Changed `params.cfg`:

```ini
[hf_endpoints]
embedding_endpoint_url = http://tei-embedding:80
reranker_endpoint_url = http://tei-reranker:80

[qdrant]
mode = native
url = http://qdrant:6333
port = 6333
collection = rag_multi_corpus
```

### Step 2: Updated Docker Compose Defaults

Changed Docker Compose defaults to local CPU-friendly TEI models:

```text
TEI_EMBEDDING_MODEL=BAAI/bge-base-en-v1.5
TEI_RERANKER_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2
```

Changed reranker healthcheck timing to give the model enough startup time.

Files changed:

```text
docker-compose/docker-compose.yml
docker-compose/.env.example
```

### Step 3: Loaded RAG-Multi-Corpus Data

We inspected the RAG-Multi-Corpus folder and found five enterprise corpora:

```text
Aventro Motors
Cendara University
Cloudway 24
Velvera Technologies
ZX Bank
```

We used prepared chunk JSONs instead of parsing raw PDF, DOCX, PPTX, HTML, and Markdown files from scratch.

The imported payload shape is:

```json
{
  "text": "chunk text",
  "metadata": {
    "enterprise_name": "ZX Bank",
    "document_source": "ZX Bank",
    "source_dataset": "RAG-Multi-Corpus",
    "filename": "Awards & Recognitions.md",
    "title": "Awards & Recognitions",
    "chunk_index": 1,
    "document_format": "md",
    "source_path": "RAG-Multi-Corpus/ZX Bank/md/Awards & Recognitions.md"
  }
}
```

Current Qdrant collection:

```text
name: rag_multi_corpus
points: 2115
vector_size: 768
distance: Cosine
status: green
```

### Step 4: Configured Enterprise Metadata Filters

Changed `params.cfg`:

```ini
[metadata_filters]
filterable_fields = enterprise_name:list
```

Changed `src/components/retriever/filters.py`:

```python
FILTER_VALUES = {
    "enterprise_name": [
        "Aventro Motors",
        "Cendara University",
        "Cloudway 24",
        "Velvera Technologies",
        "ZX Bank",
    ]
}
```

This lets the LLM extract a valid enterprise name from the query and apply it as a Qdrant filter:

```text
metadata.enterprise_name
```

### Step 5: Adjusted Generator Metadata Fields

Changed generator context metadata so answers include useful source context:

```ini
CONTEXT_META_FIELDS = filename,enterprise_name,document_source,chunk_index
TITLE_META_FIELDS = enterprise_name,filename
```

This makes source output look like:

```text
ZX Bank - Awards & Recognitions.md
```

### Step 6: Added Reranker Flat-Score Fallback

Changed `src/components/retriever/retriever_orchestrator.py`.

New behavior:

- Call reranker normally.
- Inspect reranker scores.
- If all scores are identical, treat reranking as failed.
- Fall back to vector-search order.

This improved retrieval quality for the observed local reranker behavior.

### Step 7: Added Hugging Face Router Generator Provider

Changed `src/components/utils.py`:

```python
"huggingface_router": {"api_key": os.getenv("HF_TOKEN")}
```

Changed `src/components/generator/generator_orchestrator.py`:

```python
"openai_base_url": (
    "generator",
    "OPENAI_BASE_URL",
    "GENERATOR_OPENAI_BASE_URL",
    "https://router.huggingface.co/v1",
)
```

and:

```python
"huggingface_router": lambda: ChatOpenAI(
    model=self.model,
    openai_api_key=self.auth_config["api_key"],
    base_url=self.openai_base_url,
    **common_params
)
```

Changed `params.cfg`:

```ini
PROVIDER = huggingface_router
MODEL = deepseek-ai/DeepSeek-V4-Flash:deepinfra
OPENAI_BASE_URL = https://router.huggingface.co/v1
```

### Step 8: Restarted and Smoke Tested

Restarted the ChaBo container and verified startup logs:

```text
Generator initialized with provider: huggingface_router, model: deepseek-ai/DeepSeek-V4-Flash:deepinfra
LangGraph workflow compiled
Uvicorn running on http://0.0.0.0:7860
```

Health check:

```text
GET /health -> {"status":"healthy"}
```

Full RAG smoke test:

```text
Question: What awards did ZX Bank win in 2023?
```

Result:

- Retrieved ZX Bank documents.
- Applied `enterprise_name: ZX Bank`.
- Generated answer through Hugging Face Router.
- Returned source links in the response.

## 5. Environment Variables

### 5.1 Current Required Variables

The main env file for Docker Compose is:

```text
docker-compose/.env
```

Use this template:

```text
docker-compose/.env.example
```

Current required variables:

| Variable | Current use | Source / where it came from | Secret? |
| --- | --- | --- | --- |
| `HF_TOKEN` | Used by TEI containers to download models; used by ChaBo for Hugging Face Router generation. | Provided by the user during setup. For new setups, create one in Hugging Face account settings. | Yes |
| `COMPOSE_PROFILES` | Enables local Docker services. Current intended value is `local,infra`. | Local deployment choice. Not from a third-party service. | No |
| `TEI_EMBEDDING_MODEL` | Embedding model served by local TEI. Current value: `BAAI/bge-base-en-v1.5`. | Hugging Face model ID selected for local CPU-compatible 768-dimensional embeddings. | No |
| `TEI_RERANKER_MODEL` | Reranker model served by local TEI. Current value: `cross-encoder/ms-marco-MiniLM-L-6-v2`. | Hugging Face model ID selected because it started reliably in local TEI. | No |
| `QDRANT_API_KEY` | Only needed for secured remote Qdrant. Not required for the current local Qdrant. | For Qdrant Cloud, obtain from the Qdrant Cloud console. | Yes if used |

Recommended local `docker-compose/.env` shape:

```bash
HF_TOKEN=REDACTED
QDRANT_API_KEY=
COMPOSE_PROFILES=local,infra
TEI_EMBEDDING_MODEL=BAAI/bge-base-en-v1.5
TEI_RERANKER_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2
```

Do not commit `docker-compose/.env`.

### 5.2 Generator Variables in `params.cfg`

Current generator config:

```ini
[generator]
PROVIDER = huggingface_router
MODEL = deepseek-ai/DeepSeek-V4-Flash:deepinfra
MAX_TOKENS = 1024
TEMPERATURE = 0.1
CONTEXT_META_FIELDS = filename,enterprise_name,document_source,chunk_index
TITLE_META_FIELDS = enterprise_name,filename
OPENAI_BASE_URL = https://router.huggingface.co/v1
```

The following keys remain in `params.cfg` for other providers but are not active with `PROVIDER = huggingface_router`:

```ini
INFERENCE_PROVIDER
ORGANIZATION
AZURE_ENDPOINT
```

### 5.3 Optional Provider Variables

These are only needed if switching away from the current Hugging Face Router setup:

| Variable | Use |
| --- | --- |
| `OPENAI_API_KEY` | OpenAI provider |
| `ANTHROPIC_API_KEY` | Anthropic provider |
| `COHERE_API_KEY` | Cohere provider |
| `AZURE_API_KEY` | Azure OpenAI provider |
| `GENERATOR_OPENAI_BASE_URL` | Env override for `OPENAI_BASE_URL` if using an OpenAI-compatible endpoint |

## 6. Where the Values Came From

### 6.1 Hugging Face Token

The Hugging Face token was provided by the user for this workspace. It was used for:

- TEI model download/authentication.
- Hugging Face Inference Providers Router generation.

For a new deployment, create or rotate tokens here:

```text
https://huggingface.co/settings/tokens
```

### 6.2 Hugging Face Router Base URL

The base URL came from Hugging Face Inference Providers documentation:

```text
https://router.huggingface.co/v1
```

Reference:

```text
https://huggingface.co/docs/inference-providers/index
```

### 6.3 Generator Model

The selected model/provider pair:

```text
deepseek-ai/DeepSeek-V4-Flash:deepinfra
```

was discovered from the Hugging Face Router model list and then validated with direct chat completion probes.

Other tested model/provider pairs also returned HTTP `200`, but this one produced a clean simple answer in testing.

Important caveat:

- Hugging Face free inference is credit-limited and provider/model availability can change.
- If the model stops working, query the router model list again and select a currently available chat model/provider pair.

### 6.4 Local TEI Model IDs

The embedding and reranking model IDs are Hugging Face model IDs:

```text
BAAI/bge-base-en-v1.5
cross-encoder/ms-marco-MiniLM-L-6-v2
```

These are passed into the TEI Docker image:

```yaml
image: ghcr.io/huggingface/text-embeddings-inference:cpu-1.9.2
```

TEI reference:

```text
https://huggingface.co/docs/text-embeddings-inference/index
```

### 6.5 Qdrant Values

For this local setup:

```ini
url = http://qdrant:6333
port = 6333
collection = rag_multi_corpus
```

These come from the Docker Compose service name and exposed Qdrant port.

For Qdrant Cloud, the URL and API key would come from the Qdrant Cloud console.

## 7. Current Architecture in Detail

### 7.1 Running Containers

Current relevant containers:

```text
docker-compose-chabo-1           healthy, port 7860
docker-compose-qdrant-1          running, ports 6333 and 6334
docker-compose-tei-embedding-1   healthy, port 8081 -> container port 80
docker-compose-tei-reranker-1    healthy, port 8082 -> container port 80
```


The Compose file defines `chatui`, but ChatUI is not part of the currently confirmed running service list.

### 7.2 Service-Level Architecture

```text
User / Client
  -> FastAPI + LangServe on chabo:7860
      -> /chatfed-ui-stream
      -> /chatfed-with-file-stream
      -> /health

ChaBo app
  -> LangGraph workflow
      -> ingest node
      -> extract_filters node
      -> retrieve node
      -> generate node

Retriever
  -> local TEI embedding service: http://tei-embedding:80
  -> local Qdrant: http://qdrant:6333
  -> local TEI reranker service: http://tei-reranker:80

Generator
  -> Hugging Face Router: https://router.huggingface.co/v1
  -> model: deepseek-ai/DeepSeek-V4-Flash:deepinfra
```

### 7.3 Request Flow

1. A request is sent to `/chatfed-ui-stream`.
2. `chatui_adapter` extracts the latest user message.
3. Conversation history is compacted for context.
4. LangGraph starts the workflow.
5. `ingest` optionally processes uploaded file content.
6. `extract_filters` asks the generator to identify allowed metadata filters.
7. The filter extractor maps user intent to valid values from `FILTER_VALUES`.
8. `retrieve` embeds the query with local TEI.
9. Qdrant searches `rag_multi_corpus`.
10. If a filter exists, Qdrant searches with nested payload fields such as:

```text
metadata.enterprise_name = ZX Bank
```

11. The retriever sends candidate documents to the local reranker.
12. If reranker scores are all identical, ChaBo falls back to vector-search order.
13. `generate` formats context, calls the Hugging Face Router chat model, streams tokens, and emits citations.
14. The UI adapter appends a small footnote showing applied filters and appends sources.

### 7.4 Data Architecture

Current source data:

```text
data/rag_multi_corpus/parsed-chunks-less-tokens/
```

Files:

```text
aventro_optimised_prompt_chunks.json
cendara_optimised_prompt_chunks.json
cloudway_optimised_prompt_chunks.json
velvera_optimised_prompt_chunks.json
zx_bank_optimised_prompt_chunks.json
```

Current enterprises:

```text
Aventro Motors
Cendara University
Cloudway 24
Velvera Technologies
ZX Bank
```

Qdrant collection:

```text
rag_multi_corpus
```

Payload schema pattern:

```text
payload.text
payload.metadata.enterprise_name
payload.metadata.document_source
payload.metadata.source_dataset
payload.metadata.filename
payload.metadata.title
payload.metadata.chunk_index
payload.metadata.document_format
payload.metadata.source_path
```

### 7.5 Code Architecture

#### FastAPI app

```text
src/main.py
```

Responsibilities:

- Load config.
- Parse filterable fields.
- Validate filter values.
- Initialize retriever and generator.
- Build LangGraph workflow.
- Register health and LangServe routes.

#### Workflow

```text
src/components/orchestration/workflow.py
src/components/orchestration/nodes.py
src/components/orchestration/ui_adapters.py
src/components/orchestration/state.py
```

Responsibilities:

- Define graph state.
- Build workflow nodes and edges.
- Adapt ChatUI/LangServe inputs.
- Stream generated output and source events.

#### Retriever

```text
src/components/retriever/retriever_orchestrator.py
src/components/retriever/filters.py
```

Responsibilities:

- Call embedding endpoint.
- Query Qdrant in native or Gradio mode.
- Apply metadata filters.
- Rerank candidate chunks.
- Return LangChain `Document` objects.

#### Generator

```text
src/components/generator/generator_orchestrator.py
src/components/generator/prompts.py
src/components/generator/sources.py
```

Responsibilities:

- Resolve provider/model configuration.
- Initialize LangChain chat model.
- Build RAG prompts.
- Generate streamed and non-streamed answers.
- Parse citations.
- Build source lists.

#### Ingestion

```text
src/components/ingestor/import_rag_multi_corpus.py
src/components/ingestor/upload_parquet.py
src/components/ingestor/ingestor.py
```

Responsibilities:

- Import prepared RAG-Multi-Corpus chunks.
- Embed chunks.
- Create Qdrant collection if needed.
- Upsert deterministic point IDs.
- Handle uploaded file ingestion for chat.

## 8. Current Validation Results

### 8.1 Health

```text
GET http://127.0.0.1:7860/health
```

Result:

```json
{"status":"healthy"}
```

### 8.2 Qdrant Collection

```text
GET http://127.0.0.1:6333/collections/rag_multi_corpus
```

Important result fields:

```text
status: green
points_count: 2115
vectors.size: 768
vectors.distance: Cosine
segments_count: 4
```

### 8.3 Generator Direct Test

Direct generator test with a short ZX Bank context produced:

```text
ZX Bank won the Best Digital Transformation Bank - South Asia award in 2023 [1].
```

### 8.4 Full RAG Test

Endpoint:

```text
POST http://127.0.0.1:7860/chatfed-ui-stream/invoke
```

Question:

```text
What awards did ZX Bank win in 2023?
```

Result confirmed:

- Retrieval from `rag_multi_corpus`.
- Filter extraction: `enterprise_name = ZX Bank`.
- Generation through Hugging Face Router.
- Sources returned from `ZX Bank - Awards & Recognitions.md`.

## 9. Useful Commands

### Start Full Local Stack

From the repo root:

```bash
docker compose \
  --env-file docker-compose/.env \
  -f docker-compose/docker-compose.yml \
  --profile local \
  --profile infra \
  up -d --build
```

### Check Containers

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

### Check ChaBo Health

```bash
curl -sS http://127.0.0.1:7860/health
```

### Check Qdrant Collection

```bash
curl -sS http://127.0.0.1:6333/collections/rag_multi_corpus
```

### Import RAG-Multi-Corpus

Run inside the ChaBo container:

```bash
docker exec -w /app docker-compose-chabo-1 \
  python src/components/ingestor/import_rag_multi_corpus.py \
  --collection rag_multi_corpus \
  --qdrant-url http://qdrant:6333 \
  --embedding-url http://tei-embedding:80 \
  --batch-size 32 \
  --recreate
```

Use `--limit` for a smoke import:

```bash
docker exec -w /app docker-compose-chabo-1 \
  python src/components/ingestor/import_rag_multi_corpus.py \
  --limit 20 \
  --recreate
```

### Test Full RAG Route

```bash
curl -sS -X POST http://127.0.0.1:7860/chatfed-ui-stream/invoke \
  -H 'Content-Type: application/json' \
  -d '{"input":{"messages":[{"role":"user","content":"What awards did ZX Bank win in 2023?"}]}}'
```

## 10. Known Limitations and Follow-Ups

### 10.1 Hugging Face Router Is Credit-Limited

The Hugging Face Router path avoids a dedicated paid endpoint, but it is not unlimited. Free usage depends on Hugging Face account credits and provider availability.

If generation starts failing:

1. Check the Hugging Face account billing/credits.
2. Query the router model list.
3. Select a currently available chat model/provider pair.
4. Update `MODEL` in `params.cfg`.

### 10.2 Local Reranker Quality

The current local reranker is functional but returned flat scores in testing. The fallback protects answer quality, but a stronger reranker could improve results.

Options:

- Try another TEI-supported reranker.
- Use a remote reranker endpoint.
- Disable reranking if vector ordering is consistently better.

### 10.3 Qdrant Payload Indexes

The collection currently works without payload indexes. For larger corpora, add a payload index for:

```text
metadata.enterprise_name
```

This will make filtered searches more efficient.

### 10.4 Data Directory Is Local

The RAG-Multi-Corpus chunk files are currently under:

```text
data/rag_multi_corpus/
```

If this repo is cloned elsewhere, the data must be copied or regenerated before running the import.

### 10.5 ChatUI Not Confirmed Running

The backend is healthy on port `7860`. The Compose file defines ChatUI on port `3000`, but the currently confirmed running containers did not include ChatUI.

To run it, start the full Compose stack and ensure:

```text
docker-compose/chatui.env.local
```

exists from:

```text
docker-compose/chatui.env.local.template
```

## 11. Current Changed Files

The main changes from this setup are:

```text
params.cfg
docker-compose/docker-compose.yml
docker-compose/.env.example
src/components/utils.py
src/components/generator/generator_orchestrator.py
src/components/retriever/filters.py
src/components/retriever/retriever_orchestrator.py
src/components/ingestor/import_rag_multi_corpus.py
data/rag_multi_corpus/parsed-chunks-less-tokens/*.json
docs/CHABO_PROJECT_ANALYSIS_RUNBOOK.md
```

## 12. Final Current State

The current ChaBo setup is operational as a local RAG backend:

- Backend: healthy.
- Embeddings: local TEI.
- Reranking: local TEI with flat-score fallback.
- Vector DB: local Qdrant.
- Corpus: RAG-Multi-Corpus loaded.
- Generator: Hugging Face Router using `deepseek-ai/DeepSeek-V4-Flash:deepinfra`.
- Query filtering: `enterprise_name` metadata extraction enabled.
- Smoke test: successful full RAG response with sources.
