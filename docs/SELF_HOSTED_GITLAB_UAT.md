# Self-Hosted GitLab UAT

## Current Server Setup

GitLab CE is installed on the UAT server with Docker Compose:

| Purpose | Value |
| --- | --- |
| GitLab web URL | `http://197.243.49.219:8080` |
| GitLab SSH URL | `ssh://git@197.243.49.219:2222/...` |
| GitLab data dir | `/opt/gitlab` |
| ChaBo UAT app dir | `/opt/chabo` |
| ChaBo UAT URL | `http://197.243.49.219:7860` |

The GitLab project created for this repo is:

```text
ssh://git@197.243.49.219:2222/huzalabs-ltd-1/chabo.git
```

## Why This Is Reusable

Use this GitLab instance as the deployment mirror, not necessarily the permanent source of truth.

That means the UAT deployment flow stays the same even if the ChaBo team later creates a new official GitHub repository. When that happens, point GitLab at the new upstream repository and keep deploying from GitLab `main`.

Recommended future source flow:

```text
Official ChaBo GitHub repo -> self-hosted GitLab mirror -> GitLab CI -> UAT Docker Compose
```

Local development can still push directly to this GitLab project while the official upstream does not exist. Once upstream exists, switch local work to a fork/branch workflow and let GitLab pull from or mirror the upstream repo.

## GitLab CI Architecture

The `.gitlab-ci.yml` pipeline is designed for a GitLab Runner on the same UAT server.

It does the following:

1. Builds the ChaBo Docker image locally on the UAT server.
2. Tags it as `chabo-uat:<commit-sha>`.
3. Writes `/opt/chabo/docker-compose/.env.uat` from GitLab CI variables.
4. Starts local Qdrant, TEI embedding, TEI reranker, and ChaBo with Docker Compose.
5. Imports RAG-Multi-Corpus only when the Qdrant collection is empty.

This avoids needing a container registry for UAT. The compose file supports this through:

```text
CHABO_PULL_POLICY=never
```

## Required GitLab CI Variables

Set these in the GitLab project under **Settings -> CI/CD -> Variables**:

| Variable | Required | Notes |
| --- | --- | --- |
| `HF_TOKEN` | Yes | Hugging Face token used by TEI and Hugging Face Router. Mask this variable. |
| `QDRANT_API_KEY` | No | Leave empty for local unsecured UAT Qdrant. |
| `GENERATOR_MODEL` | No | Defaults to `deepseek-ai/DeepSeek-V4-Flash:deepinfra`. |
| `GENERATOR_OPENAI_BASE_URL` | No | Defaults to `https://router.huggingface.co/v1`. |
| `IMPORT_RAG_MULTI_CORPUS` | No | Defaults to `true`. |

## Runner Requirements

The GitLab Runner should be registered with tag:

```text
uat
```

It must be able to run Docker commands against the host Docker daemon and write to `/opt/chabo`.

Recommended runner shape:

```text
GitLab Runner container
  -> Docker executor
  -> /var/run/docker.sock mounted
  -> /opt/chabo mounted
```

## When Official ChaBo GitHub Exists

Do this instead of rebuilding the deployment setup:

1. Add the new GitHub repo as the upstream source.
2. Configure the self-hosted GitLab project as a pull mirror of that repo, or manually sync with `git fetch upstream`.
3. Keep `.gitlab-ci.yml`, Docker Compose, and UAT variables in the GitLab deployment branch.
4. Deploy from GitLab as before.

If upstream does not accept our deployment files, keep a small `uat` branch in GitLab that rebases or merges from upstream `main` and retains only deployment-specific files.

## Useful Commands

```bash
git remote add huzalabs ssh://git@197.243.49.219:2222/huzalabs-ltd-1/chabo.git
git push huzalabs main
```

```bash
ssh -p 5050 joel@197.243.49.219
cd /opt/gitlab
sudo docker compose ps
```

```bash
cd /opt/chabo
docker compose --env-file docker-compose/.env.uat -f docker-compose/docker-compose.uat.yml ps
```
