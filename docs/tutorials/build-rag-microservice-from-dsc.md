# Build a RAG Microservice from the DSC ChaBo Code

Related issue: [#24 Document how to build a RAG microservice from the existing DSC chabo codes](https://github.com/ChaBo-Project/chabo-microservices/issues/24)

This guide explains how the current DSC ChaBo codebase can be turned into a
repeatable RAG microservice setup. It focuses on the parts that matter when
creating a new ChaBo implementation from this baseline: runtime services,
configuration, data import, update flow, and Docker Compose operations.

## What This Baseline Contains

The current baseline is a FastAPI RAG orchestrator packaged as the `chabo`
service. It is not only a demo chatbot; it is the working reference for a
RAG-backed ChaBo implementation.

The baseline includes:

| Area | Current implementation |
| --- | --- |
| API runtime | FastAPI app in `src/main.py`. |
| RAG workflow | LangGraph workflow in `src/components/orchestration/`. |
| Retriever | Qdrant-backed retriever in `src/components/retriever/`. |
| Embedding and reranking | Hugging Face TEI endpoints, local by default in Docker Compose. |
| Generator | Configurable provider in `src/components/generator/`, currently Hugging Face Router by default. |
| Data import | RAG-Multi-Corpus importer and parquet uploader under `src/components/ingestor/`. |
| Channels | Optional speech and WhatsApp channel microservices under `services/`. |
| Deployment | Local and UAT Docker Compose files under `docker-compose/`. |

## Target Architecture

For a normal RAG chatbot implementation, run these services:

```text
User / Channel
  -> ChaBo orchestrator API
  -> TEI embedding service
  -> Qdrant vector database
  -> TEI reranker service
  -> LLM provider
  -> ChaBo response
```

Optional channels, such as WhatsApp or speech, sit in front of the orchestrator:

```text
WhatsApp or speech channel
  -> normalized channel event
  -> ChaBo orchestrator
  -> RAG answer
  -> channel response
```

## Repository Layout

The most important paths are:

| Path | Purpose |
| --- | --- |
| `src/main.py` | Starts the ChaBo orchestrator API. |
| `params.cfg` | Main RAG configuration: endpoints, Qdrant collection, retrieval, generator, metadata filters, and chunking. |
| `docker-compose/docker-compose.yml` | Local development stack. |
| `docker-compose/docker-compose.uat.yml` | UAT stack using prebuilt images. |
| `docker-compose/.env.example` | Local environment template. |
| `docker-compose/.env.uat.example` | UAT environment template. |
| `scripts/deploy_uat.sh` | Builds/pushes the backend image and deploys the UAT compose files. |
| `src/components/ingestor/import_rag_multi_corpus.py` | Imports the bundled RAG-Multi-Corpus JSON chunks into Qdrant using TEI embeddings. |
| `src/components/ingestor/upload_parquet.py` | Imports a parquet dataset with precomputed vectors. |
| `services/speech-channel/` | Optional speech channel microservice. |
| `services/whatsapp-channel/` | Optional WhatsApp channel microservice. |

## Configuration Model

ChaBo uses two configuration layers.

### `params.cfg`

`params.cfg` controls application behavior inside the backend image:

```ini
[hf_endpoints]
embedding_endpoint_url = http://tei-embedding:80
reranker_endpoint_url = http://tei-reranker:80

[qdrant]
mode = native
url = http://qdrant:6333
collection = rag_multi_corpus

[retrieval]
initial_k = 20
final_k = 5

[generator]
PROVIDER = huggingface_router
MODEL = deepseek-ai/DeepSeek-V4-Flash:deepinfra
OPENAI_BASE_URL = https://router.huggingface.co/v1

[metadata_filters]
filterable_fields = enterprise_name:list
```

Change `params.cfg` when a new implementation needs:

- A different Qdrant collection name.
- Different embedding or reranker endpoints.
- Different retrieval depth.
- A different LLM provider or model.
- Different metadata fields for filtering and citations.
- Different chunking settings.

### Compose `.env` Files

Compose environment files hold deployment values and secrets:

| File | Use |
| --- | --- |
| `docker-compose/.env` | Local development secrets and profile settings. |
| `docker-compose/.env.uat` | UAT runtime secrets and image tags. |

Create the local file from the template:

```bash
cp docker-compose/.env.example docker-compose/.env
```

Create the UAT file on the server from the template:

```bash
cp docker-compose/.env.uat.example docker-compose/.env.uat
```

Do not commit `.env`, `.env.uat`, tokens, or app secrets.

## Local Development Setup

Use this path when adapting the DSC baseline into a new RAG microservice.

### 1. Clone and select the development branch

```bash
git clone https://github.com/ChaBo-Project/chabo-microservices.git
cd chabo-microservices
git checkout development
```

### 2. Create local environment files

```bash
cp docker-compose/.env.example docker-compose/.env
cp docker-compose/chatui.env.local.template docker-compose/chatui.env.local
```

Set at minimum:

```ini
HF_TOKEN=...
QDRANT_API_KEY=
COMPOSE_PROFILES=local,infra
TEI_EMBEDDING_MODEL=BAAI/bge-base-en-v1.5
TEI_RERANKER_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2
```

Leave `QDRANT_API_KEY` empty for the local unsecured Qdrant service.

### 3. Start the self-hosted local stack

```bash
docker compose \
  --env-file docker-compose/.env \
  -f docker-compose/docker-compose.yml \
  --profile local \
  --profile infra \
  up -d --build
```

This starts:

- `chabo` on `http://localhost:7860`
- ChatUI on `http://localhost:3000`
- Qdrant on `http://localhost:6333`
- TEI embedding on `http://localhost:8081`
- TEI reranker on `http://localhost:8082`

If you use a remote Qdrant instead, update `params.cfg [qdrant]` and omit the
`infra` profile:

```bash
docker compose \
  --env-file docker-compose/.env \
  -f docker-compose/docker-compose.yml \
  --profile local \
  up -d --build
```

### 4. Import RAG data

The current DSC baseline uses RAG-Multi-Corpus chunks under:

```text
data/rag_multi_corpus/parsed-chunks-less-tokens/
```

Import them with:

```bash
docker compose \
  --env-file docker-compose/.env \
  -f docker-compose/docker-compose.yml \
  exec chabo python src/components/ingestor/import_rag_multi_corpus.py \
    --collection rag_multi_corpus \
    --qdrant-url http://qdrant:6333 \
    --embedding-url http://tei-embedding:80 \
    --batch-size 32 \
    --recreate
```

The importer:

- Reads `*_optimised_prompt_chunks.json` files.
- Infers an `enterprise_name` from the filename prefix.
- Embeds chunk text with TEI.
- Creates a Qdrant collection with the correct vector size.
- Stores payloads as `{ "text": "...", "metadata": {...} }`.

For a different dataset, either adapt `import_rag_multi_corpus.py` or create a
new importer that produces the same Qdrant payload shape.

### 5. Check the API

```bash
curl http://localhost:7860/health
```

Open:

```text
http://localhost:7860/docs
```

Useful endpoints:

| Endpoint | Purpose |
| --- | --- |
| `/health` | Runtime health check. |
| `/chatfed-ui-stream/invoke` | LangServe endpoint used by ChatUI and channel services. |
| `/v1/speech/*` | Built-in speech endpoints on the orchestrator. |

## Building a New RAG Implementation

To create a new RAG implementation from the DSC baseline:

1. Choose the dataset and target domain.
2. Decide whether the implementation belongs in this repo or in a demo/application repo.
3. Create or adapt an importer for the dataset.
4. Configure the Qdrant collection in `params.cfg`.
5. Configure metadata fields and filter values.
6. Choose embedding and reranking models.
7. Choose the generator provider and model.
8. Run locally with Docker Compose.
9. Import data into Qdrant.
10. Test retrieval and answer quality.
11. Build and deploy an image to UAT.

### Data Shape

ChaBo expects each Qdrant point payload to contain:

```json
{
  "text": "chunk text",
  "metadata": {
    "enterprise_name": "Aventro Motors",
    "filename": "example.md",
    "document_source": "Aventro Motors",
    "chunk_index": 1
  }
}
```

The `text` field is used for context. The `metadata` object is used for
citations, titles, and optional metadata filters.

### Metadata Filters

If `params.cfg` contains:

```ini
[metadata_filters]
filterable_fields = enterprise_name:list
```

then `src/components/retriever/filters.py` must define allowed values for
`enterprise_name`. ChaBo validates this at startup. If a field is declared in
`params.cfg` but missing from `filters.py`, the service will refuse to start.

Use metadata filters only for fields that are stable, useful to users, and
actually present in the Qdrant payload.

## Pulling Updates

Use `development` as the UAT baseline branch.

```bash
git checkout development
git pull origin development
```

If the remote is named `microservices` locally:

```bash
git fetch microservices
git checkout development
git merge --ff-only microservices/development
```

Feature work should happen on issue-linked branches:

```bash
git checkout -b docs/example-issue-24
```

After review, merge PRs into `development`. UAT should pull or deploy from
`development`, not from short-lived feature branches.

## Docker Compose Management

### Local stack

Start:

```bash
docker compose \
  --env-file docker-compose/.env \
  -f docker-compose/docker-compose.yml \
  --profile local \
  --profile infra \
  up -d --build
```

Check:

```bash
docker compose \
  --env-file docker-compose/.env \
  -f docker-compose/docker-compose.yml \
  ps
```

Logs:

```bash
docker compose \
  --env-file docker-compose/.env \
  -f docker-compose/docker-compose.yml \
  logs -f chabo
```

Stop:

```bash
docker compose \
  --env-file docker-compose/.env \
  -f docker-compose/docker-compose.yml \
  down
```

### UAT stack

The UAT stack uses prebuilt images. The main file is:

```text
docker-compose/docker-compose.uat.yml
```

Required UAT values:

```ini
COMPOSE_PROJECT_NAME=chabo-uat
CHABO_IMAGE=...
HF_TOKEN=...
TEI_EMBEDDING_MODEL=BAAI/bge-base-en-v1.5
TEI_RERANKER_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2
GENERATOR_PROVIDER=huggingface_router
GENERATOR_MODEL=deepseek-ai/DeepSeek-V4-Flash:deepinfra
GENERATOR_OPENAI_BASE_URL=https://router.huggingface.co/v1
IMPORT_RAG_MULTI_CORPUS=true
```

Deploy using:

```bash
CHABO_IMAGE=registry.example.com/chabo/chabo:uat \
UAT_HOST=uat.example.com \
UAT_USER=joel \
UAT_SSH_PORT=5050 \
UAT_APP_DIR=/home/joel/chabo \
scripts/deploy_uat.sh
```

The script:

1. Builds the backend image.
2. Pushes the image to the configured registry.
3. Uploads UAT compose files.
4. Updates `CHABO_IMAGE` in the remote `.env.uat`.
5. Runs `docker compose pull` and `docker compose up -d`.
6. Waits for `/health`.
7. Imports RAG-Multi-Corpus if the collection is empty and `IMPORT_RAG_MULTI_CORPUS=true`.

Set `IMPORT_RAG_MULTI_CORPUS=false` after data is already loaded and you only
want to deploy code updates.

## Updating a Running UAT Deployment

Use this flow for normal code changes:

1. Merge the PR into `development`.
2. Build and push a new backend image.
3. Deploy the image with `scripts/deploy_uat.sh`.
4. Confirm health:

```bash
curl https://chabo.giz.huzalabs.com/health
```

5. Check logs on the UAT server:

```bash
docker compose \
  --env-file docker-compose/.env.uat \
  -f docker-compose/docker-compose.uat.yml \
  logs -f chabo
```

Use this flow for data changes:

1. Add or update dataset files.
2. Rebuild the backend image if the data is bundled in the image.
3. Set `IMPORT_RAG_MULTI_CORPUS=true` for a first import or empty collection.
4. For a forced rebuild of the collection, run the importer manually with
   `--recreate`.

## When to Create a Separate Demo Repository

The microservices repository should hold reusable services and shared docs. A
domain-specific dummy chatbot should move to a demo/application repo when it
contains:

- Domain-specific data.
- Domain-specific prompts or UI copy.
- Demo-only Docker Compose wiring.
- A complete example bot that consumes reusable ChaBo microservices.

For example, a future `chabo-demo/example-bot` can contain:

- The example orchestrator configuration.
- Demo dataset import scripts.
- Demo compose file that pulls reusable microservice images.
- Demo instructions for ChatUI, WhatsApp, and speech channels.

Keep reusable code, shared channel services, generic API standards, and shared
deployment patterns in `chabo-microservices`.

## Common Problems

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Backend starts but answers without context | Qdrant collection is empty or collection name is wrong. | Check `params.cfg [qdrant] collection` and run the importer. |
| Qdrant search fails because of vector size | Collection was created with a different embedding model. | Recreate the collection with embeddings from the intended model. |
| Service fails at startup with metadata filter error | `filterable_fields` and `filters.py` do not match. | Add allowed values in `src/components/retriever/filters.py` or remove the field from `params.cfg`. |
| TEI containers stay unhealthy | Model download is still running, token is missing, or architecture is unsupported. | Check TEI logs and `HF_TOKEN`; first startup can take several minutes. |
| UAT deploy updates code but data remains old | Existing Qdrant volume was reused. | Run importer manually with `--recreate` if a full data refresh is intended. |
| ChatUI returns origin or cookie errors on HTTP | SvelteKit origin/cookie settings are too strict for non-HTTPS. | Set `UI_ORIGIN` and allow insecure cookies only for non-HTTPS test deployments. |

## Minimal Definition of Done

A new RAG microservice implementation is ready for review when:

- It runs locally with Docker Compose.
- `/health` returns healthy.
- Qdrant contains the expected collection and point count.
- A representative question returns cited context from the intended dataset.
- Metadata filters either work or are intentionally disabled.
- Runtime secrets live in `.env` or server secret storage, not in Git.
- UAT deploy instructions and environment values are documented.
- Any domain-specific demo assets are clearly separated from reusable
  microservice code.
