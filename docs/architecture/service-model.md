# Service Model

ChaBo services should be designed as small, documented, independently deployable
microservices.

## Expected Service Properties

- Containerized with Docker.
- Exposes a clear HTTP API or event interface.
- Provides health and readiness endpoints.
- Reads configuration from environment variables.
- Avoids hard-coded secrets.
- Includes usage documentation and example requests.
- Can be deployed independently or through Docker Compose.

## Recommended Service Categories

- Channel services: website widget, WhatsApp, speech, Facebook Messenger.
- AI services: NLU, dialog management, LLM + RAG, voice technologies.
- Management modules: user management, LLM management, dialog management.
- Infrastructure services: logging, usage statistics, monitoring.

## Minimum API Expectations

Every service should document:

- Base URL and route list.
- Request and response schemas.
- Authentication requirements.
- Required environment variables.
- Error responses.
- Health check endpoint.
