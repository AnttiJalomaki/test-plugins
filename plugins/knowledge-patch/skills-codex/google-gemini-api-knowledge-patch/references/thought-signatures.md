# Thought signatures and history replay

Source batch: `gemini-3-thought-signatures`.

Use this reference when an application stores content parts itself, implements a
function loop, consumes a stream, or adapts tool calls to a compatibility API.

## Preserve signatures on their original parts

Thought signatures are opaque encrypted reasoning state returned on response
parts. Official SDKs preserve them when the complete response object is appended
to history. REST clients and manually assembled histories must return every
signature unchanged on the exact model part where it arrived.

```json
{
  "role": "model",
  "parts": [{
    "functionCall": {"name": "check_flight", "args": {"flight": "AA100"}},
    "thoughtSignature": "<opaque signature>"
  }]
}
```

For Gemini 3.x function calling, this round trip is mandatory even with minimal
thinking. Omitting a required signature produces HTTP 400.

## Current-turn validation and sequential calls

Validation scans backward to the newest user message containing ordinary
content. A user message whose only content is `functionResponse` does not start
a new turn. Every step after the boundary must retain the signature on its first
function call, so a sequential loop resends all earlier signed model-call parts:

```text
user(text) -> model(FC1 + signature A) -> user(FR1)
           -> model(FC2 + signature B) -> user(FR2)
```

Do not discard the first signed model call after receiving and answering a
second call in the same turn.

## Keep parallel calls grouped

When a model response contains parallel calls, only its first `functionCall`
part carries the signature. Leave the signature on that part, then replay every
model call together followed by every function response:

```text
model: [FC1 + signature, FC2]
user:  [FR1, FR2]
```

Interleaving `FC1, FR1, FC2, FR2` fails validation.

## OpenAI-compatible envelope

Chat-completion responses store the signature on the signed tool call at
`extra_content.google.thought_signature`. Preserve the extension when replaying
the assistant tool-call message.

```json
{
  "tool_calls": [{
    "extra_content": {
      "google": {"thought_signature": "<opaque signature>"}
    },
    "function": {
      "name": "check_flight",
      "arguments": "{\"flight\":\"AA100\"}"
    }
  }]
}
```

## Non-call and streamed signatures

Without a function call, Gemini 3.x may attach a signature to the last content
part. Returning it is recommended for reasoning continuity but is not validated.
In a streamed non-call response the signature may arrive on a part whose text is
empty. Consume through `finish_reason`; do not stop merely because a part has no
text.

The series differ:

- With function calls, Gemini 2.5 places an optional signature on the first part
  regardless of type. Gemini 3.x signs the first function-call part and requires
  it back.
- Without function calls, Gemini 2.5 returns no signature. Gemini 3.x can sign
  the last part after generating a thought.

## Imported unsigned traces

Avoid injecting function-call blocks that the API did not produce. When a trace
must be imported and therefore cannot contain real signatures, either documented
sentinel can bypass validation in the signature field:

```json
{"thoughtSignature": "context_engineering_is_the_way_to_go"}
```

```json
{"thoughtSignature": "skip_thought_signature_validator"}
```

Use sentinels only for imported unsigned traces, never as a replacement for a
real signature that the API returned.
