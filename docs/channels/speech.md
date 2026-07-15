# Speech Channel

The speech channel is a reusable microservice for voice-based chatbot
interactions.

Related issue: [#6 Create speech channel](https://github.com/ChaBo-Project/chabo-microservices/issues/6)

## Provider Choice

The first cloud provider target is Google Cloud:

- Speech-to-Text for English speech recognition.
- Text-to-Speech for answer audio generation.
- Python client libraries for both directions.
- Application Default Credentials for local development and deployment without
  committing service-account keys.

The service also includes a `mock` provider. This is the default for local demos
and automated tests, because it does not require cloud credentials or billable
speech API calls.

Official references:

- [Google Cloud Speech-to-Text client libraries](https://cloud.google.com/speech-to-text/docs/transcribe-client-libraries)
- [Google Cloud Text-to-Speech client libraries](https://cloud.google.com/text-to-speech/docs/create-audio-text-client-libraries)

## API Location

```text
src/components/speech/
```

The first implementation is mounted into the current ChaBo FastAPI backend, so
the speech APIs appear in the existing API docs:

```text
http://localhost:7860/docs
```

The repository also keeps a standalone service skeleton at:

```text
services/speech-channel/
```

## What Is Needed For It To Work

The speech channel can run in two modes.

| Mode | Purpose | External dependency |
| --- | --- | --- |
| `mock` | Local development, automated tests, and first UAT smoke demo. | None. |
| `google` | Real English speech-to-text and text-to-speech. | Google Cloud Speech-to-Text, Google Cloud Text-to-Speech, and runtime credentials. |

For the first working demo, use `SPEECH_PROVIDER=mock`. This proves the channel
contract, the ChaBo handoff, container startup, routing, and documentation
without waiting for cloud account setup.

For a real speech demo, the runtime needs:

- A Google Cloud project with billing enabled.
- Speech-to-Text API enabled.
- Text-to-Speech API enabled.
- A service account or workload identity that can call both APIs.
- Runtime credentials provided through Application Default Credentials.
- `SPEECH_PROVIDER=google`.
- `GOOGLE_CLOUD_PROJECT` or `GOOGLE_PROJECT_ID`.

Credentials must be configured in the deployment environment. They must not be
committed to the repository.

## API

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/v1/speech/health` | Speech API health and active provider. |
| `GET` | `/v1/speech/provider` | Provider, language, voice, and encoding metadata. |
| `POST` | `/v1/speech/transcribe` | Accepts an uploaded audio file and returns text. |
| `POST` | `/v1/speech/synthesize` | Accepts text and returns base64 audio. |
| `POST` | `/v1/speech/chat` | Transcribes audio, calls the current ChaBo workflow, and optionally returns synthesized answer audio. |

## Local Demo

Run the normal ChaBo backend. The speech endpoints are exposed on the same
backend port as the existing API.

```bash
python -u src/main.py
```

The backend is exposed locally on port `7860`.

```bash
curl -F "audio=@question.wav" http://localhost:7860/v1/speech/transcribe
```

With the mock provider, a UTF-8 text file can be used instead of real audio:

```bash
printf "What is ChaBo?" > question.txt
curl -F "audio=@question.txt" http://localhost:7860/v1/speech/transcribe
```

## UAT Rollout

The recommended UAT rollout is:

1. Merge the speech channel branch into `development`.
2. Deploy the updated ChaBo backend through the existing UAT workflow.
3. Keep the speech endpoints under the existing HTTPS backend route.
4. Start with `SPEECH_PROVIDER=mock` and verify the channel endpoints.
5. Add Google Cloud credentials only after the mock path is working.
6. Add a small browser demo that records microphone audio, sends it to
   `/v1/speech/chat`, and plays the returned answer audio.

The integrated speech API calls the existing ChaBo workflow in process:

```text
speech upload -> speech provider -> ChaBo workflow -> optional answer audio
```

The browser or external client should call only the HTTPS route exposed by Nginx.

## Configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `SPEECH_PROVIDER` | `mock` | `mock` or `google`. |
| `SPEECH_LANGUAGE_CODE` | `en-US` | Recognition and synthesis language. |
| `SPEECH_VOICE_NAME` | `en-US-Standard-J` | Google Text-to-Speech voice name. |
| `SPEECH_AUDIO_ENCODING` | `MP3` | Google Text-to-Speech output encoding. |
| `GOOGLE_CLOUD_PROJECT` | unset | Required for `SPEECH_PROVIDER=google`. |
| `GOOGLE_SPEECH_LOCATION` | `global` | Google Speech-to-Text recognizer location. |
| `GOOGLE_SPEECH_RECOGNIZER_ID` | `_` | Google Speech-to-Text recognizer identifier. |
| `GOOGLE_SPEECH_STT_MODEL` | `latest_short` | Google Speech-to-Text model. |

## Implementation Notes

- The service is request/response first. Streaming audio can be added later
  without changing the basic channel contract.
- `/v1/speech/chat` maps one uploaded utterance to one ChaBo user message.
- `session_id` is accepted as form data, passed as workflow metadata, and echoed
  back for client correlation.
- Audio content should not be logged by default.
- Cloud credentials must be provided by the runtime environment, not committed
  to the repository.

## Remaining Work

- Deploy the updated backend to UAT behind HTTPS.
- Add a small browser or mobile demo client for recording audio.
- Validate real Google Cloud credentials and English audio samples.
- Decide whether to add streaming speech recognition for longer conversations.
