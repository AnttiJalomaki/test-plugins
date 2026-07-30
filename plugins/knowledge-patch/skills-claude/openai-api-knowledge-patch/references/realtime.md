# Realtime API

Use this reference to choose a dedicated Realtime model and endpoint, implement continuous translation, transport safety identifiers, mint client credentials, and update beta-shaped clients to GA session and event names.

## Choose the workflow and endpoint

| Workflow | Model | Endpoint or behavior |
| --- | --- | --- |
| Voice agent with responses, tools, and conversation state | `gpt-realtime-2.1` | `/v1/realtime` |
| Continuous speech translation | `gpt-realtime-translate` | `/v1/realtime/translations` |
| Live transcript deltas | `gpt-realtime-whisper` | Dedicated transcript-delta workflow |

Realtime 2 voice sessions expose `reasoning.effort`.

Do not route translation through the normal voice-agent turn lifecycle merely because both use streaming audio. Translation has its own session behavior.

## Continuous translation lifecycle

Translation sessions stream translated audio and transcript deltas continuously. They do not use the normal assistant-turn lifecycle.

- Do not send `response.create` to start each translated response.
- Do not wait for the client to commit a user turn before translation begins.

Build consumers around continuous output deltas rather than a sequence of committed user turns and created assistant responses.

## Safety identifier transport

Realtime uses the `OpenAI-Safety-Identifier` header. Responses instead uses the `safety_identifier` request parameter; that parameter is not carried into Realtime automatically.

Set the Realtime header at the trusted point that creates or owns the session:

- Put it on the server-side request that mints a client secret to bind an ephemeral session.
- Or put it on a trusted server's WebSocket or unified WebRTC connection.

The identifier does not carry over from Responses requests or from other Realtime sessions. Use a stable, privacy-preserving value for the same end user while explicitly attaching it to each applicable session or trusted connection.

## Browser and mobile credentials

Browser and mobile applications obtain ephemeral client credentials from:

```text
POST /v1/realtime/client_secrets
```

Attach the safety header to the server-side client-secret request when binding the identifier to that session.

## GA WebRTC setup

Use the released WebRTC call endpoint:

```text
/v1/realtime/calls
```

The removed beta interface used different shapes. Updating only the URL or removing `OpenAI-Beta: realtime=v1` is insufficient when the client still emits beta session fields or listens for beta event names.

## GA session configuration

GA session configuration sets `session.type`. Output-audio configuration is nested below:

```text
session.audio.output
```

Update serializers and generated types that still expect beta-era top-level output-audio fields.

## GA response deltas

Listen for the released event names:

- `response.output_text.delta`
- `response.output_audio.delta`
- `response.output_audio_transcript.delta`

Route each event by its payload type. Do not wait for a text event when the selected workflow primarily emits audio or transcript deltas.

## Retirement alignment

The `OpenAI-Beta: realtime=v1` interface was removed May 12, 2026. For model-family migration scheduled January 20, 2027:

- `gpt-realtime` and GPT-4o realtime families move to `gpt-realtime-2.1`.
- Realtime mini families move to `gpt-realtime-2.1-mini`.
- GPT audio and GPT-4o audio families move to `gpt-audio-1.5`.
- `gpt-4o-mini-transcribe-2025-03-20` moves to `gpt-4o-mini-transcribe-2025-12-15`.

See [endpoint-state-and-lifecycle.md](endpoint-state-and-lifecycle.md) for the full retirement map.

## Client migration checklist

1. Classify the workload as voice agent, continuous translation, or transcription.
2. Select its dedicated model and endpoint.
3. Replace the removed beta header and migrate all session and event shapes.
4. Mint ephemeral credentials server-side for browser and mobile clients.
5. Attach `OpenAI-Safety-Identifier` at the correct trusted session boundary.
6. Set `session.type` and nest output audio under `session.audio.output`.
7. Consume the GA text, audio, and audio-transcript delta event names.
8. Test translation without `response.create` or user-turn commit gates.
