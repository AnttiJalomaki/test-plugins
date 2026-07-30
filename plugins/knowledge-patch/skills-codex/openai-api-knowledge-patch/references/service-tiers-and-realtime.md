# Service Tiers and Realtime

## Flex processing

Flex applies to Responses and Chat Completions at Batch API token rates.
Prompt-cache discounts remain available.

Official SDK requests default to a ten-minute timeout and automatically retry
`408 Request Timeout` twice. Long Flex work can therefore require a larger
client-level or per-request timeout:

```python
response = client.with_options(timeout=900.0).responses.create(
    model="<supported-model>",
    input="<long-running task>",
    service_tier="flex",
)
```

A Flex capacity shortage returns `429 Resource Unavailable` and does not charge
the request. Retry with exponential backoff to preserve Flex pricing. To fall
back to the project's normal processing mode, retry with
`service_tier="auto"` or omit the field.

## Priority processing

Set `service_tier="priority"` per request, or configure the project to use
Priority when requests omit the field. A project-level change is applied
gradually. Inspect the response `service_tier` to determine the tier that
actually handled any request.

```json
{
  "model": "<supported-model>",
  "input": "Latency-sensitive request",
  "service_tier": "priority"
}
```

Standard and Priority share the same per-model rate limit. At one million TPM
or more, increasing traffic by over 50 percent within 15 minutes can trigger
the ramp limit. An affected Priority request is processed with
`service_tier: "default"` and billed at Standard rates. Ramp sustained traffic
gradually.

Priority preserves prompt-cache discounts and supports multimodal image input.
It does not support long-context requests, fine-tuned models, or embeddings.
Because it charges a per-token premium, reserve it for steady,
latency-sensitive traffic rather than erratic batch or evaluation workloads.

## Realtime workload selection

Use the dedicated realtime model and endpoint for each lifecycle:

- Use `gpt-realtime-2.1` on `/v1/realtime` for a voice agent that responds,
  calls tools, and retains conversation state. Realtime 2 voice sessions expose
  `reasoning.effort`.
- Use `gpt-realtime-translate` on `/v1/realtime/translations` for continuous
  translation.
- Use `gpt-realtime-whisper` for live transcript-delta workflows.

## Continuous translation lifecycle

A translation session streams translated audio and transcript deltas without
the usual assistant-turn lifecycle. Do not send `response.create`. Do not wait
for the client to commit a user turn before translation starts.

## Realtime safety identifiers

Realtime accepts the `OpenAI-Safety-Identifier` header, not the Responses
`safety_identifier` request parameter.

Place the header on the server-side client-secret request to bind it to an
ephemeral session. Alternatively, set it on a trusted server's WebSocket or
unified WebRTC connection. It does not carry over from a Responses request or
from another realtime session.

## Client credentials and WebRTC

Have browser and mobile clients obtain ephemeral credentials through:

```text
POST /v1/realtime/client_secrets
```

Use the GA WebRTC endpoint:

```text
/v1/realtime/calls
```

## GA session and event shapes

Set `session.type` in the GA session configuration. Nest output-audio settings
under `session.audio.output`.

Update streaming handlers to the GA event names:

- `response.output_text.delta`
- `response.output_audio.delta`
- `response.output_audio_transcript.delta`
