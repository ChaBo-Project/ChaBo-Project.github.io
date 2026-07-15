# UAT Deployment

The current UAT deployment runs ChaBo behind Nginx with HTTPS.

## Public Endpoint

```text
https://chabo.giz.huzalabs.com/docs
```

## Current Runtime Services

- ChaBo FastAPI/LangServe service.
- Local Qdrant vector database.
- Local Hugging Face TEI embedding service.
- Local Hugging Face TEI reranker service.
- Nginx reverse proxy with Let us Encrypt TLS.

## Planned Speech API

The next UAT expansion is the speech API inside the current ChaBo backend. The
first deployment should run it with the mock provider so that routing,
container startup, and the ChaBo handoff can be tested without Google Cloud
credentials.

Target shape:

```text
browser demo or API client
  -> https://chabo.giz.huzalabs.com/v1/speech/
  -> Nginx
  -> chabo container
  -> in-process ChaBo workflow
```

After the mock flow is verified, the same service can be switched to
`SPEECH_PROVIDER=google` by adding Google Cloud runtime credentials and project
configuration to the UAT environment.

## Notes

This UAT environment demonstrates the current RAG chatbot baseline. It is not
yet the final microservices architecture.
