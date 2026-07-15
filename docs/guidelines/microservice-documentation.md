# Microservice Documentation Guidelines

Every reusable ChaBo microservice should include enough documentation for another
team to run, test, and integrate it without private knowledge.

## Required Sections

Each service page should include:

- Purpose: what the service does and when to use it.
- Architecture: where it fits in the ChaBo ecosystem.
- API: endpoints, schemas, and examples.
- Configuration: environment variables and defaults.
- Deployment: Docker image, Docker Compose example, and runtime requirements.
- Security: authentication, secrets, and webhook validation.
- Operations: health checks, logs, metrics, and troubleshooting.
- Contribution notes: known gaps and future improvements.

## Service Page Template

```markdown
# Service Name

## Purpose

## Architecture

## API

## Configuration

## Deployment

## Security

## Operations

## Troubleshooting
```

## Documentation Quality Bar

- Keep setup steps executable.
- Prefer examples over abstract descriptions.
- Mark secrets clearly and never commit real values.
- Include local development and production notes separately.
- Link related GitHub issues and pull requests.
