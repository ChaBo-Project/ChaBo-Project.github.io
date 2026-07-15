# ChaBo UAT Deployment: Option 2, Prebuilt Image + Registry

This is the recommended UAT deployment path when we want to push updates from this workspace to a UAT server in a controlled way.

The model is:

```text
Local machine or CI
  -> build ChaBo backend image
  -> push image to registry
  -> SSH into UAT
  -> pull image
  -> restart Docker Compose stack
  -> verify health
```

The UAT server runs:

- ChaBo backend from the prebuilt image.
- Local Qdrant with a persistent volume.
- Local TEI embedding service.
- Local TEI reranker service.

The UAT server does not bind-mount the source repo. Code and bundled corpus files come from the Docker image.

## Files Added for UAT

```text
.github/workflows/deploy-uat.yml
docker-compose/docker-compose.uat.yml
docker-compose/.env.uat.example
scripts/deploy_uat.sh
.dockerignore
```

## GitHub Actions Deployment

The repository now has a UAT deployment workflow:

```text
.github/workflows/deploy-uat.yml
```

It runs automatically when code is pushed to `main`, and it can also be run manually from:

```text
GitHub -> Actions -> Build and Deploy UAT -> Run workflow
```

The workflow:

1. Builds the ChaBo backend image.
2. Pushes it to GitHub Container Registry.
3. Uploads the UAT compose/env files to the UAT server over SSH.
4. Writes `/opt/chabo/docker-compose/.env.uat` from GitHub secrets and variables.
5. Pulls the image on UAT.
6. Starts/restarts the UAT Docker Compose stack.
7. Waits for `/health`.
8. Imports RAG-Multi-Corpus if enabled and the Qdrant collection is empty.

Default image name:

```text
ghcr.io/chabo-project/chabo:uat-<short-git-sha>
```

The repo name is lowercased automatically because Docker image references must be lowercase.

## GitHub Secrets

Add these in:

```text
GitHub -> Settings -> Secrets and variables -> Actions -> Secrets
```

Required:

| Secret | Description |
| --- | --- |
| `UAT_HOST` | UAT server hostname or IP. |
| `UAT_USER` | SSH username on the UAT server. |
| `UAT_SSH_KEY` | Private SSH key that can connect to the UAT server. |
| `HF_TOKEN` | Hugging Face token used by TEI and Hugging Face Router generation. |

Optional:

| Secret | Description |
| --- | --- |
| `UAT_SSH_PORT` | SSH port. Defaults to `22`. |
| `QDRANT_API_KEY` | Leave unset/empty for local unsecured UAT Qdrant. |
| `GHCR_PULL_USERNAME` | GHCR username for private package pull on UAT. Defaults to the GitHub actor if `GHCR_PULL_TOKEN` is set. |
| `GHCR_PULL_TOKEN` | GitHub PAT with `read:packages`, needed if the GHCR package is private and UAT is not already logged in. |

For `UAT_SSH_KEY`, create a deploy key or server-specific SSH key. The public key goes into the UAT user's `~/.ssh/authorized_keys`; the private key goes into this GitHub secret.

## GitHub Variables

Add these in:

```text
GitHub -> Settings -> Secrets and variables -> Actions -> Variables
```

All variables are optional because the workflow has defaults.

| Variable | Default | Description |
| --- | --- | --- |
| `UAT_APP_DIR` | `/opt/chabo` | Directory on UAT where compose files live. |
| `UAT_COMPOSE_PROJECT_NAME` | `chabo-uat` | Docker Compose project name. |
| `CHABO_BIND` | `0.0.0.0` | Bind address for ChaBo. Use `127.0.0.1` behind a reverse proxy. |
| `CHABO_PORT` | `7860` | Host port for ChaBo. |
| `TEI_EMBEDDING_MODEL` | `BAAI/bge-base-en-v1.5` | Local TEI embedding model. |
| `TEI_RERANKER_MODEL` | `cross-encoder/ms-marco-MiniLM-L-6-v2` | Local TEI reranker model. |
| `GENERATOR_PROVIDER` | `huggingface_router` | ChaBo generator provider override. |
| `GENERATOR_MODEL` | `deepseek-ai/DeepSeek-V4-Flash:deepinfra` | Generator model/provider pair. |
| `GENERATOR_OPENAI_BASE_URL` | `https://router.huggingface.co/v1` | Hugging Face Router OpenAI-compatible base URL. |
| `IMPORT_RAG_MULTI_CORPUS` | `true` | Whether to import corpus when Qdrant is empty on push-triggered deploys. |

## GitHub Actions First-Time UAT Setup

On the UAT server:

```bash
sudo mkdir -p /opt/chabo
sudo chown -R "$USER":"$USER" /opt/chabo
docker --version
docker compose version
```

If the GHCR image package is private, either:

- Add `GHCR_PULL_USERNAME` and `GHCR_PULL_TOKEN` secrets, or
- Log into GHCR once on UAT:

```bash
docker login ghcr.io
```

Then trigger the workflow manually from GitHub Actions.

Manual inputs:

| Input | Use |
| --- | --- |
| `image_tag` | Optional. Example: `uat-2026-05-20-001`. If empty, the workflow uses `uat-<short-sha>`. |
| `import_rag_multi_corpus` | Keep true on the first deploy. Later it is safe either way because import is skipped if points already exist. |

## GitHub Actions Update Flow

After the first successful deploy, normal updates are:

```bash
git add .
git commit -m "Update ChaBo UAT deployment"
git push origin main
```

GitHub Actions will build, push, deploy, health-check, and report status.

## One-Time UAT Server Requirements

Install on the UAT server:

- Docker Engine.
- Docker Compose plugin: `docker compose`.
- `curl`.
- SSH access from this machine or CI.

Open the backend port if users will call ChaBo directly:

```text
7860/tcp
```

If using a reverse proxy such as Nginx, keep ChaBo bound to `127.0.0.1` and expose the proxy instead.

## Registry Requirements

Use any Docker registry your team can access:

- GHCR.
- Docker Hub.
- GitLab Container Registry.
- AWS ECR.
- Azure Container Registry.
- A private registry.

Both the local machine/CI and the UAT server must be logged into the registry.

Example:

```bash
docker login ghcr.io
```

Do this locally for `docker push`, and on the UAT server for `docker pull`.

## First-Time Deploy

From the project root on this machine:

```bash
export CHABO_IMAGE=ghcr.io/YOUR_ORG/chabo:uat
export UAT_HOST=YOUR_UAT_HOST_OR_IP
export UAT_USER=YOUR_SSH_USER
export UAT_APP_DIR=/opt/chabo

scripts/deploy_uat.sh
```

On the first run, the script uploads the UAT compose files and creates:

```text
/opt/chabo/docker-compose/.env.uat
```

Then it stops and asks you to fill in the UAT values.

SSH into UAT:

```bash
ssh YOUR_SSH_USER@YOUR_UAT_HOST_OR_IP
cd /opt/chabo
nano docker-compose/.env.uat
```

Minimum values:

```bash
COMPOSE_PROJECT_NAME=chabo-uat
CHABO_IMAGE=ghcr.io/YOUR_ORG/chabo:uat
CHABO_BIND=0.0.0.0
CHABO_PORT=7860
HF_TOKEN=REDACTED
QDRANT_API_KEY=
TEI_EMBEDDING_MODEL=BAAI/bge-base-en-v1.5
TEI_RERANKER_MODEL=cross-encoder/ms-marco-MiniLM-L-6-v2
GENERATOR_PROVIDER=huggingface_router
GENERATOR_MODEL=deepseek-ai/DeepSeek-V4-Flash:deepinfra
GENERATOR_OPENAI_BASE_URL=https://router.huggingface.co/v1
IMPORT_RAG_MULTI_CORPUS=true
```

Do not commit `.env.uat`.

Then rerun from this machine:

```bash
scripts/deploy_uat.sh
```

## What the Deploy Script Does

`scripts/deploy_uat.sh`:

1. Builds the backend image from `docker-compose/backend.Dockerfile`.
2. Tags it with `CHABO_IMAGE`.
3. Pushes it to the registry.
4. Copies `docker-compose.uat.yml` and `.env.uat.example` to the UAT server.
5. Updates `CHABO_IMAGE` in the UAT `.env.uat` file to match the image tag being deployed.
6. Runs `docker compose pull` on UAT.
7. Runs `docker compose up -d` on UAT.
8. Waits for `http://127.0.0.1:7860/health`.
9. Imports RAG-Multi-Corpus if `IMPORT_RAG_MULTI_CORPUS=true` and the Qdrant collection is empty.
10. Prints the final Compose status.

## Normal Update Flow

After UAT is configured, pushing updates is this:

```bash
export CHABO_IMAGE=ghcr.io/YOUR_ORG/chabo:uat
export UAT_HOST=YOUR_UAT_HOST_OR_IP
export UAT_USER=YOUR_SSH_USER
export UAT_APP_DIR=/opt/chabo

scripts/deploy_uat.sh
```

This rebuilds the image with the latest local code, pushes it, pulls it on UAT, and restarts the stack.

If the corpus is already imported, set this on the UAT server:

```bash
IMPORT_RAG_MULTI_CORPUS=false
```

That makes later deployments faster.

## Versioned Tags

For cleaner rollback, prefer immutable tags. The deploy script updates `CHABO_IMAGE` automatically, so rerunning with a new tag is enough:

```bash
export CHABO_IMAGE=ghcr.io/YOUR_ORG/chabo:uat-2026-05-20-001
scripts/deploy_uat.sh
```

For convenience, you can still keep a moving `:uat` tag, but immutable tags are safer for UAT signoff.

## Rollback

On UAT:

```bash
cd /opt/chabo
nano docker-compose/.env.uat
```

Set `CHABO_IMAGE` to the previous known-good tag, then run:

```bash
docker compose \
  --env-file docker-compose/.env.uat \
  -f docker-compose/docker-compose.uat.yml \
  pull chabo

docker compose \
  --env-file docker-compose/.env.uat \
  -f docker-compose/docker-compose.uat.yml \
  up -d chabo
```

## Health Check

From the UAT server:

```bash
curl -sS http://127.0.0.1:7860/health
```

Expected:

```json
{"status":"healthy"}
```

From your machine, if the port is open:

```bash
curl -sS http://YOUR_UAT_HOST_OR_IP:7860/health
```

## Full RAG Smoke Test

From the UAT server:

```bash
curl -sS -X POST http://127.0.0.1:7860/chatfed-ui-stream/invoke \
  -H 'Content-Type: application/json' \
  -d '{"input":{"messages":[{"role":"user","content":"What awards did ZX Bank win in 2023?"}]}}'
```

Expected behavior:

- It returns a generated answer.
- It applies `enterprise_name: ZX Bank`.
- It returns sources from `ZX Bank - Awards & Recognitions.md`.

## UAT Architecture

```text
User or tester
  -> UAT server:7860
      -> chabo container
          -> tei-embedding container
          -> qdrant container
          -> tei-reranker container
          -> Hugging Face Router for generation
```

Only ChaBo needs to be reachable from outside the server. Qdrant and TEI can remain internal Docker services.

## Data Persistence

Qdrant data is stored in a Docker volume:

```text
chabo-uat_qdrant_data
```

TEI model caches are stored in Docker volumes:

```text
chabo-uat_tei-embedding-data
chabo-uat_tei-reranker-data
```

Do not delete these volumes unless you intentionally want to reload models and reimport data.

## Notes

- The Hugging Face token must stay in `/opt/chabo/docker-compose/.env.uat` or a server-side secret manager.
- The Docker image includes the application code, `params.cfg`, import script, and current `data/rag_multi_corpus` JSON files.
- If the image gets too large later, move corpus files to a separate artifact or object storage and import them from there.
- If UAT has limited CPU/RAM, consider using remote embedding/reranker endpoints instead of local TEI.
