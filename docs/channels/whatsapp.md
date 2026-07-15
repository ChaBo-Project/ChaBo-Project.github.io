# WhatsApp Channel

The WhatsApp channel will allow ChaBo implementations to receive and respond to
messages through WhatsApp.

Related issue: [#3 Implement a WhatsApp channel](https://github.com/ChaBo-Project/chabo-microservices/issues/3)

## Current Scope

The first implementation provides a reusable WhatsApp channel microservice with
mock-mode tests and configurable Meta credentials. Mock mode allows local
development and CI validation before the Meta app, phone number, and webhook
configuration are available.

See [WhatsApp Meta Setup Research](whatsapp-meta-setup.md) for the current
registration and implementation plan.

## Expected Service Responsibilities

- Receive webhook events from Meta.
- Validate webhook verification requests.
- Verify message signatures when configured.
- Normalize incoming messages into a ChaBo channel event.
- Forward events to the chatbot orchestration service.
- Send responses back through the WhatsApp Cloud API.
- Log delivery errors without exposing user content or secrets.

## Required Configuration

The service supports the following environment variables:

| Variable | Purpose |
| --- | --- |
| `WHATSAPP_PROVIDER` | `mock` for local tests, `meta` for Meta Cloud API. |
| `WHATSAPP_VERIFY_TOKEN` | Token used during webhook verification. |
| `WHATSAPP_ACCESS_TOKEN` | Meta API token for sending messages. |
| `WHATSAPP_PHONE_NUMBER_ID` | WhatsApp Business phone number identifier. |
| `WHATSAPP_APP_SECRET` | Optional app secret for signature validation. |
| `WHATSAPP_API_VERSION` | Graph API version, default `v20.0`. |
| `WHATSAPP_GRAPH_API_BASE_URL` | Graph API base URL, default `https://graph.facebook.com`. |
| `CHABO_ORCHESTRATOR_URL` | URL of the ChaBo chatbot/orchestrator service. |
| `CHABO_CHAT_PATH` | Chat endpoint path, default `/chatfed-ui-stream/invoke`. |

## Local Mock Run

```bash
cd services/whatsapp-channel
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
WHATSAPP_PROVIDER=mock \
WHATSAPP_VERIFY_TOKEN=local-verify-token \
uvicorn app.main:app --reload --port 8084
```

Verify the webhook challenge:

```bash
curl "http://localhost:8084/webhook?hub.mode=subscribe&hub.verify_token=local-verify-token&hub.challenge=12345"
```

Send a mock outbound message:

```bash
curl -X POST http://localhost:8084/messages/send \
  -H "Content-Type: application/json" \
  -d '{"to":"250788000000","text":"Hello from ChaBo"}'
```

## Open Decisions

- Whether the first UAT demo should expose `/whatsapp/webhook` or a separate
  WhatsApp subdomain.
- How to persist conversation/session state.
- How the service should handle media messages.
- Which production Meta Business account and phone number should own the
  channel.
