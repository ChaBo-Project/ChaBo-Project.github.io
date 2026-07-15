# Ecosystem Overview

The ChaBo ecosystem is organized around a shared core repository and reusable
services that implementers can reuse or extend.

## Core Repository

The core repository provides:

- Microservice source code.
- Example integrations.
- Documentation.
- Issue tracking.
- Reusable deployment patterns.

## Implementations

Implementers build chatbot solutions by combining:

- Existing ChaBo services.
- New implementation-specific services.
- Channel adapters such as website, WhatsApp, or speech.
- Management modules for users, dialogs, LLMs, and usage insights.

## Contribution Loop

New technical development should be contributed back when it is reusable:

1. Build a service or integration in an implementation project.
2. Document the service interface, configuration, and deployment.
3. Contribute the reusable part back through a pull request.
4. Publish container images when the service is ready for reuse.
