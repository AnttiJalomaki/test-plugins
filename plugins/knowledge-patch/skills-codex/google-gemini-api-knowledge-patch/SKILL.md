---
name: google-gemini-api-knowledge-patch
description: Google Gemini API
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Google Gemini API Knowledge Patch

Use this skill when implementing, migrating, or operating Gemini API clients.
Start with the breaking-change checks below, then open the topic reference that
matches the task. Treat concrete model aliases, quotas, and lifecycle dates as
operational inputs that should be checked before deployment.

## Reference index

| Reference | Topics |
| --- | --- |
| [thought-signatures.md](references/thought-signatures.md) | Signature preservation, turn validation, sequential and parallel calls, compatibility envelopes, streaming, imported traces |
| [interactions-api.md](references/interactions-api.md) | `steps` migration, stateless history, SSE assembly, continuation, response formats, background agents |
| [models-and-lifecycle.md](references/models-and-lifecycle.md) | Production model behavior, removed controls, endpoint lifecycle, multimodal services, Live sessions, long-running work |
| [sdk-migration.md](references/sdk-migration.md) | GA package names, centralized clients, per-call config, async access, JavaScript response shapes, automatic functions, parsing, caches, embeddings |
| [function-calling-and-tools.md](references/function-calling-and-tools.md) | Function declarations, tool choice, identity, multimodal results, MCP, streamed arguments, pre-tool text |
| [structured-outputs.md](references/structured-outputs.md) | Recursive schemas, stream accumulation, built-in tools, supported schema subset |
| [billing-and-limits.md](references/billing-and-limits.md) | Billing tiers, Prepay and Postpay, spend caps, quota dimensions, spend-rate enforcement, Priority and Batch pools |

## Breaking-change triage

Before changing application logic, identify the API surface:

- `generateContent` uses content parts and preserves `thoughtSignature` on the
  exact model part that received it.
- Interactions uses typed `steps`, top-level `response_format`, and
  `previous_interaction_id` plus `function_result` for continuation.
- The OpenAI-compatible chat surface carries a signed tool call under
  `extra_content.google.thought_signature`.

Do not translate field names mechanically between these surfaces.

### Preserve Gemini 3.x thought signatures

For manually managed history, replay each opaque signature unchanged on its
original model part. A Gemini 3.x function call requires the signature even at
minimal thinking and returns HTTP 400 when it is missing.

For sequential calls in one turn, retain all earlier signed model-call parts.
A user message containing only a function response does not create a new turn.
For parallel calls, keep all model calls together and all function responses
together; only the first function call in the model response is signed.

```text
model: [FC1 + signature, FC2]
user:  [FR1, FR2]
```

Read [thought-signatures.md](references/thought-signatures.md) before assembling
history manually, adapting chat-completion messages, or consuming signatures
from streams.

### Migrate Interactions clients to typed steps

Current Interactions responses use `steps`, not flat `outputs`. Model content
is nested in `model_output`; thoughts, client function calls, and server-side
tool calls/results are separate typed steps. `POST /interactions` returns output
steps, while `GET /interactions/{id}` returns the full timeline.

For stateless continuation, pass the prior `steps` array as the next `input`
and append a `user_input` step. For client-side functions, wait for
`requires_action`, execute the call, then continue with the previous interaction
ID and matching call ID. Server-side tools do not require that round trip.

Streaming clients must assemble indexed steps from:

```text
interaction.created
step.start -> step.delta -> step.stop
interaction.completed
[DONE]
```

The completion event has status and usage but no steps. Preserve unknown event
and delta variants by logging and skipping them rather than failing.

### Update Interactions output configuration

Use top-level, type-discriminated `response_format`. Structured JSON uses a
text format with `mime_type` and `schema`; image settings belong in an image
format; audio uses an audio format instead of `response_modalities`. Supply an
array for multiple modalities and accept interleaved deltas.

```python
response_format={
    "type": "text",
    "mime_type": "application/json",
    "schema": Result.model_json_schema(),
}
```

Read [interactions-api.md](references/interactions-api.md) for the migration
dates, raw event shapes, multimodal format examples, and background-agent flow.

### Remove rejected or ignored generation patterns

On the current production Flash family:

- remove `temperature`, `top_p`, and `top_k`; they are ignored now and future
  generations will reject them;
- replace `thinking_budget` with string-valued `thinking_level` when moving to
  Gemini 3.6 Flash;
- remove `candidate_count` for Gemini 3.x;
- never end a request with a non-empty `model` turn as a completion prefill;
  use `system_instruction` or `response_format` instead;
- include both `call_id` and function `name` on every Gemini 3.x
  `FunctionResponse` sent through `generateContent`.

Pin a concrete model ID when reproducibility matters. Moving aliases have
changed targets more than once, and published shutdown dates require planned
migrations. See [models-and-lifecycle.md](references/models-and-lifecycle.md).

## GA SDK migration quick reference

Use the current packages and service-oriented clients:

| Language | Package | Generation entry point |
| --- | --- | --- |
| Python | `google-genai` | `client.models.generate_content(...)` |
| JavaScript | `@google/genai` | `ai.models.generateContent(...)` |
| Go | `google.golang.org/genai` | `client.Models.GenerateContent(...)` |

Put optional generation settings in each call's `config`; Python async methods
live under `client.aio`. In JavaScript, read `response.text` as a property and
iterate the object returned by `generateContentStream` directly.

Python callables passed as tools execute automatically by default. Disable
automatic function calling when the application must authorize, inspect, or
dispatch the call itself. Pydantic response schemas expose validated output at
`response.parsed`. Caches are created through `client.caches` and referenced by
name from generation config. JavaScript embeddings are returned as the plural
`result.embeddings`.

See [sdk-migration.md](references/sdk-migration.md) for complete code patterns.

## Function-calling quick reference

Interactions custom functions are direct entries in `tools`:

```python
tool = {
    "type": "function",
    "name": "get_weather",
    "description": "Get weather for a city.",
    "parameters": {
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"],
    },
}
```

Tool choice supports `auto`, `any`, `none`, and preview `validated`; restrict
the callable set through nested `allowed_tools`. Large or deeply nested schemas
can be rejected. A function result may contain multiple typed text and image
blocks, but must preserve its name and call ID.

Remote MCP tools use `type: mcp_server`; they support Streamable HTTP only,
require names without hyphens, and may carry headers and an allowed-tools list.
For streamed calls, capture ID and name at `step.start`, accumulate arguments
by step index, and parse JSON only after completion.

Read [function-calling-and-tools.md](references/function-calling-and-tools.md)
before building a manual dispatcher or combining client and server tools.

## Structured-output quick reference

Recursive response schemas can use `$ref: "#"` for the root. Structured stream
fragments are partial JSON text: concatenate them in order and validate only
after the document is complete. Gemini 3-series Interactions can, in preview,
run built-in tools and constrain the final text with a schema in the same
request.

The supported subset includes nullable unions, schema- or boolean-valued
`additionalProperties`, date/time string formats, numeric bounds, and array
`prefixItems`, `minItems`, and `maxItems`. See
[structured-outputs.md](references/structured-outputs.md).

## Billing and quota guardrails

Billing plan, usage tier, and account spend cap belong to the Cloud Billing
account and are inherited by linked projects. Request quotas remain per project,
shared by all keys. A zero prepaid balance or account cap can stop every linked
project; neither is an API-key setting.

Enforce client backoff against all independent interactive dimensions: RPM,
input TPM, and RPD. A paid project can also hit a rolling spend-rate limit and
receive `429 RESOURCE_EXHAUSTED` while token/request capacity remains. HTTP 400
and 500 requests consume quota even though their tokens are not billed.

Priority inference has a separate pool but also counts toward overall
interactive traffic. Batch traffic is isolated from non-batch quota and has its
own concurrency, storage, and per-model enqueued-token ceilings. Read
[billing-and-limits.md](references/billing-and-limits.md) before capacity or
cost planning.

## Implementation checklist

1. Identify `generateContent`, Interactions, or the compatibility surface.
2. Pin the intended model ID and check its lifecycle deadline.
3. Remove unsupported generation controls and model-prefill turns.
4. Preserve thought signatures, call IDs, function names, and step ordering.
5. Accumulate streamed text or arguments before parsing.
6. Validate schemas against the supported subset and keep them compact.
7. Confirm project quota plus billing-account plan, balance, and spend caps.
8. Expect aliases, preview endpoints, quotas, and account status to move.
