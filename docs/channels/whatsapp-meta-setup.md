# WhatsApp Meta Setup Research

Related issue: [#3 Implement a WhatsApp channel](https://github.com/ChaBo-Project/chabo-microservices/issues/3)

## Objective

Define the first implementation path for a reusable WhatsApp channel
microservice in the ChaBo ecosystem.

The first implementation can run in `mock` mode before Meta assets are
available. Switching to Meta Cloud API only requires changing environment
values.

## Meta Assets Required

The WhatsApp channel needs the following Meta-side assets:

| Asset | Purpose | Owner |
| --- | --- | --- |
| Meta developer account | Access Meta developer tools and app dashboard. | Implementer |
| Meta app | Hosts the WhatsApp product integration and webhook settings. | Implementer |
| WhatsApp Business Account | Business container for phone numbers and templates. | Organization |
| Test or production phone number | Sends and receives WhatsApp messages. | Organization |
| Phone number ID | Used by the Cloud API send-message endpoint. | Organization |
| WhatsApp access token | Authorizes outgoing API calls. | Organization |
| Webhook callback URL | Public HTTPS endpoint exposed by the WhatsApp channel service. | DevOps |
| Webhook verify token | Shared secret used during webhook verification. | DevOps |
| App secret | Used for webhook signature validation when enabled. | Organization |

## Registration Checklist

1. Confirm which Meta Business account should own the WhatsApp integration.
2. Create or select a Meta app for the ChaBo WhatsApp channel.
3. Add the WhatsApp product to the Meta app.
4. Configure a test phone number for development.
5. Capture the phone number ID and WhatsApp Business Account ID.
6. Generate a temporary token for development testing.
7. Plan a production token strategy with a system user or approved equivalent.
8. Deploy a public HTTPS callback URL for the webhook service.
9. Configure webhook callback URL and verify token in the Meta app.
10. Subscribe to message webhook events.
11. Send a test inbound message and confirm webhook delivery.
12. Send a test outbound response through the Cloud API.

## Proposed Message Flow

```text
WhatsApp user
  -> Meta WhatsApp Cloud API webhook
  -> whatsapp-channel service
  -> ChaBo orchestrator/chatbot endpoint
  -> whatsapp-channel service
  -> Meta send-message API
  -> WhatsApp user
```

## Proposed Service API

The WhatsApp channel service should expose:

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/webhook` | Meta webhook verification challenge. |
| `POST` | `/webhook` | Receive WhatsApp webhook events. |
| `GET` | `/health` | Health check for deployment and monitoring. |
| `POST` | `/messages/send` | Optional internal endpoint for sending outbound messages. |

## Normalized Channel Event

Incoming WhatsApp payloads should be normalized before forwarding to ChaBo:

```json
{
  "channel": "whatsapp",
  "provider": "meta-cloud-api",
  "conversation_id": "whatsapp:+2507xxxxxxx",
  "user_id": "+2507xxxxxxx",
  "message_id": "wamid...",
  "message_type": "text",
  "text": "Hello",
  "metadata": {
    "phone_number_id": "1234567890",
    "wa_business_account_id": "1234567890"
  }
}
```

## Configuration

Expected environment variables:

| Variable | Required | Purpose |
| --- | --- | --- |
| `WHATSAPP_PROVIDER` | Yes | `mock` for local tests, `meta` for Meta Cloud API. |
| `WHATSAPP_VERIFY_TOKEN` | Yes | Token used for webhook verification. |
| `WHATSAPP_ACCESS_TOKEN` | When `meta` | Token used to call Meta Cloud API. |
| `WHATSAPP_PHONE_NUMBER_ID` | When `meta` | Phone number ID used to send messages. |
| `WHATSAPP_APP_SECRET` | Recommended | App secret used for webhook signature validation. |
| `WHATSAPP_API_VERSION` | No | Meta Graph API version, for example `v20.0`. |
| `CHABO_ORCHESTRATOR_URL` | Yes | ChaBo endpoint that receives normalized events. |
| `CHANNEL_LOG_LEVEL` | No | Service log level. |

## Security Requirements

- Do not commit access tokens or app secrets.
- Store secrets in GitHub Actions, GitLab CI variables, or server secret storage.
- Validate the webhook verification token.
- Validate `X-Hub-Signature-256` when an app secret is configured.
- Log message IDs and delivery states, but avoid logging sensitive message bodies in production.
- Restrict internal send-message endpoints if they are exposed.

## Local Development Approach

For local development:

1. Run the WhatsApp channel service locally.
2. Expose it through a temporary HTTPS tunnel.
3. Configure the tunnel URL as the Meta webhook callback.
4. Use a Meta test phone number and allowlisted recipient number.
5. Forward normalized events to a local or UAT ChaBo orchestrator.

## First Implementation Tasks

1. Create a minimal FastAPI service for webhook verification and message receipt. Done for mock/Meta-ready MVP.
2. Add request models for webhook payloads. Done for text messages.
3. Normalize text messages into the shared channel event format. Done.
4. Forward normalized events to the ChaBo orchestrator. Done, with mock fallback when no orchestrator URL is set.
5. Send text responses through the WhatsApp Cloud API. Done behind `WHATSAPP_PROVIDER=meta`; mock sender is available for tests.
6. Add Dockerfile and Docker Compose example. Done for the optional `channels` profile.
7. Add tests for webhook verification, signature validation, and message normalization. Done for the MVP path.
8. Document setup, configuration, and troubleshooting. In progress; Meta dashboard screenshots and production webhook notes remain.

## Open Questions

- Which Meta Business account will own the production WhatsApp number?
- Do we have an approved phone number, or should we start with a test number?
- What orchestrator endpoint should the first channel service call?
- Should media messages be supported in the first version?
- Should the channel service store conversation state or remain stateless?
