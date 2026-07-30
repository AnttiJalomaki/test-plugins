---
name: google-gemini-api-knowledge-patch
description: Google Gemini API
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Google Gemini API Knowledge Patch

Use this skill when implementing or migrating Gemini API applications,
especially Interactions, function calling, structured output, streaming,
multimodal generation, SDK clients, or quota and billing controls. Start with
the breaking changes below, then open only the reference needed for the task.

## Reference index

| Reference | Topics |
| --- | --- |
| [Interactions API](references/interactions-api.md) | Steps schema, stateless history, SSE assembly, function continuation, response formats, background agents |
| [Function calling and thought signatures](references/function-calling-and-signatures.md) | Signature replay, sequential and parallel calls, tool declarations and choice, multimodal results, remote MCP |
| [Models, media, and lifecycle](references/models-media-and-lifecycle.md) | Production IDs, thinking controls, endpoint migrations, image, video, embeddings, Live, files, long-running work |
| [Google Gen AI SDK migration](references/genai-sdk-migration.md) | Current packages and clients, per-call config, async Python, JavaScript response shapes, caches, embeddings |
| [Structured outputs](references/structured-outputs.md) | Recursive schemas, streamed JSON assembly, built-in tools, supported schema features |
| [Limits and billing](references/limits-and-billing.md) | Account tiers and caps, Prepay/Postpay, project limits, quotas, spend rate, priority and batch pools |

## Start with required migrations

### Use the current SDK packages

Replace the legacy libraries with the centralized Google Gen AI SDK:

| Language | Current package |
| --- | --- |
| Python | `google-genai` |
| JavaScript/TypeScript | `@google/genai` |
| Go | `google.golang.org/genai` |

Create one client and use its `models`, `files`, `caches`, and other services.
Put generation options in each call's `config`; do not keep them on a model
object. In Python, asynchronous methods are under `client.aio`. In JavaScript,
read `response.text` directly and iterate the value returned by
`generateContentStream`.

### Migrate Interactions consumers to typed steps

Interactions responses use `steps`, not flat `outputs`. Model content is a
`model_output` step; thoughts, function calls, and server-side calls and
results are separate typed steps. A create response contains output steps,
while a get response contains the full timeline, including `user_input`.

For stateless continuation, submit all preceding steps as the next `input`,
then append a new `user_input`. For client-executed functions, continue with
`previous_interaction_id` and a `function_result` containing the matching
function name and call ID.

The current stream is:

```text
interaction.created
  step.start → step.delta* → step.stop
  ...
interaction.completed
[DONE]
```

The completion event has status and usage but no assembled steps. Build each
step from its indexed events, tolerate unknown event variants, and do not parse
partial function arguments or structured JSON until all fragments arrive.

### Remove unsupported generation patterns

For current 3-series endpoints:

- Remove `candidate_count`.
- Replace `thinking_budget` with string-valued `thinking_level` where required.
- Remove `temperature`, `top_p`, and `top_k` from 3.6 requests; they are ignored
  now and later generations are expected to reject them.
- Do not end a request with a non-empty `model` turn. Put format and
  preamble-suppression instructions in `system_instruction` or
  `response_format`.
- With legacy `generateContent`, include both `call_id` and function `name` in
  every 3.x `FunctionResponse`.

## Preserve thought signatures exactly

Thought signatures are opaque. If history is manually assembled, replay each
signature unchanged on the exact model part where it arrived. Official SDKs
preserve this when the complete response object is appended to history.

For 3.x function calling, the first function-call part of every step after the
latest ordinary user message must retain its signature. A user message that
contains only a function response does not begin a new turn. Sequential loops
therefore resend all earlier signed model calls.

Parallel calls from one model response must remain grouped:

```text
model: [FC1 + signature, FC2]
user:  [FR1, FR2]
```

Do not interleave calls and results. In OpenAI-compatible chat messages, retain
`tool_calls[].extra_content.google.thought_signature`.

Consume a non-call stream through `finish_reason`, even when a part has empty
text, because a signature may be attached there. Non-call 3.x signatures are
recommended for continuity but are not validated.

## Continue streamed function calls safely

On `step.start`, record the step index, call ID, and function name. Append
argument fragments for that index and decode JSON only after the step ends or
the interaction completes. The revisioned wire schema and SDK event objects
name these fields differently; use the concrete event types provided by the
chosen client. See the Interactions and function-calling references for both
shapes.

If the interaction ends in `requires_action`, run only the client-side
functions and submit their results in a new interaction. Server-side tools run
without a client round trip and appear as call/result step pairs.

Declare Interactions functions directly in `tools`:

```python
tools=[{
    "type": "function",
    "name": "get_weather",
    "description": "Get weather for a city.",
    "parameters": {
        "type": "object",
        "properties": {"city": {"type": "string"}},
        "required": ["city"],
    },
}]
```

Tool choice is configured through `generation_config.tool_choice`. Use `auto`,
`any`, `none`, or the preview `validated` mode, and use nested `allowed_tools`
to restrict callable functions. Large or deeply nested schemas may be rejected.

## Select stable endpoints deliberately

Pin a concrete endpoint whenever reproducibility matters; aliases can move
between previews and GA releases. The primary production Interactions IDs are
`gemini-3.6-flash` and `gemini-3.5-flash-lite`. Their default thinking levels
are `medium` and `minimal`; both support a 1M-token context window, 64k-token
maximum output, and native Computer Use.

Before deployment, inspect the lifecycle reference for scheduled shutdowns.
Several embedding, image, 2.5-series, and Flash-Lite migrations have fixed
deadlines. Do not send traffic to the retired Gemini 2.0 stable endpoints or
the retired image and Veo generation IDs.

Treat preview capabilities as preview:

- Computer Use on `gemini-3.5-flash`
- built-in tools combined with a structured final response
- current Live audio-to-audio and TTS endpoints
- `gemini-omni-flash-preview` conversational video generation

## Configure response formats by output type

Interactions uses top-level, discriminated `response_format` entries. For
structured text, set `type: text`, `mime_type: application/json`, and `schema`.
Image settings belong in an image entry, and audio uses `type: audio`. Supply
an array for multiple modalities and expect their stream deltas to interleave.

Structured streams deliver partial JSON in text deltas. Concatenate in order,
then validate the complete document. Recursive schemas may use `$ref: "#"` to
refer to the root. Supported features include nullable unions,
`additionalProperties`, date/time formats, numeric bounds, and tuple-like
`prefixItems`, but schema size and nesting limits still apply.

In Python, a Pydantic class supplied as `response_schema` is validated and
returned as `response.parsed`.

## Control automatic Python function execution

Passing a Python callable in `tools` enables automatic execution by default.
Disable it when the application must authorize, inspect, or dispatch the call:

```python
config=types.GenerateContentConfig(
    tools=[get_current_weather],
    automatic_function_calling={"disable": True},
)
```

After disabling it, inspect the returned `function_call` part and perform the
dispatch explicitly.

## Guard production billing and quotas

Billing plan, usage tier, and account spend cap belong to the Cloud Billing
account and affect every linked project. Request quotas are per project, so all
API keys in that project share counters. A depleted prepaid balance or account
cap can stop every linked project.

Do not treat project caps as immediate circuit breakers. Billing and cap
processing can lag by roughly ten minutes, and long-running batches or agents
can continue spending during the delay. Use application-side budgets and
cancellation in addition to AI Studio caps.

Handle each enforcement dimension independently:

- requests per minute
- input tokens per minute
- requests per day
- paid-tier rolling spend rate
- priority inference pool
- batch enqueued-token pool

A `429 RESOURCE_EXHAUSTED` can be caused by spend rate even when RPM and TPM
remain. Failed HTTP 400 and 500 requests consume quota even though their tokens
are not billed. `GetTokens` is neither billed nor charged to inference quota.

## Operational checklist

- Pin concrete model IDs and review their shutdown dates.
- Return opaque signatures on their original parts.
- Keep parallel calls grouped and preserve every call ID and function name.
- Assemble indexed stream steps and tolerate unknown event variants.
- Parse argument and JSON fragments only after completion.
- Separate client-side functions from server-executed tools.
- Confirm SDK response shapes before porting legacy accessors.
- Verify billing-account state, project quotas, spend rate, and batch limits.
