---
name: anthropic-api-knowledge-patch
description: Anthropic API
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Anthropic API Compatibility Guide

Use this skill when implementing or reviewing Messages API integrations,
model migrations, tools, streaming, structured output, prompt caching, rate
limits, hosted-platform behavior, or Managed Agents.

Treat live model metadata, request validation, response fields, and project
tests as authoritative. API behavior differs by model, surface, workspace,
and enabled beta, so do not infer one target's contract from another.

## Reference index

| Reference | Topics |
| --- | --- |
| [models-and-migrations.md](references/models-and-migrations.md) | Model IDs, Claude 5 migration, thinking, sampling, lifecycle, token budgets, images |
| [structured-output.md](references/structured-output.md) | JSON output, parse helpers, strict tools, schema limits, compliance and data handling |
| [tools-and-streaming.md](references/tools-and-streaming.md) | Eager tool input, stream aggregation and recovery, beta headers, fallback events, hosted tools |
| [prompt-caching.md](references/prompt-caching.md) | Automatic and explicit caching, TTLs, invalidation, pre-warming, isolation, diagnosis |
| [platforms-and-limits.md](references/platforms-and-limits.md) | Rate and spend limits, AWS surfaces, WIF, inference geography, API discovery, tunnels |
| [managed-agents.md](references/managed-agents.md) | Agents, sessions, event streams, memory, secrets, schedules, webhooks, enterprise administration |

## Breaking migrations first

### Moving to Claude 5

- Remove assistant-message prefills.
- Omit non-default `temperature`, `top_p`, and `top_k`; use effort and prompt
  instructions instead.
- Replace top-level `output_format` with `output_config.format` for raw
  requests. The Python `messages.parse()` convenience API is the exception.
- Default to `thinking: {"type": "adaptive"}`. Do not send manual
  `budget_tokens` to Claude 5 targets.
- Request readable thinking with `display: "summarized"`; otherwise it is not
  returned.
- Keep `max_tokens` large enough for thinking, tool work, and visible output;
  it remains the hard combined ceiling.
- Preserve thinking blocks unchanged only when continuing with the same
  model. Strip `thinking` and `redacted_thinking` before switching models.
- Recount tokens and retune output and compaction budgets for each target.
- Handle `model_context_window_exceeded` separately from `max_tokens`.

```python
response = client.messages.create(
    model=model_id,
    max_tokens=128000,
    thinking={"type": "adaptive", "display": "summarized"},
    output_config={"effort": "high", "format": output_format},
    messages=messages,
)
```

### Target-specific thinking

- Fable 5 and Mythos 5 cannot disable thinking.
- Opus 5 may disable thinking only at `high`, `medium`, or `low`; `xhigh` and
  `max` return HTTP 400.
- Sonnet 5 may disable thinking at every effort level.
- Disabling thinking on Opus 5 can expose tool calls as text or internal XML,
  so validate visible output before using this mode.
- Haiku 4.5 rejects adaptive thinking and retains optional manual extended
  thinking.

### Remove retired switches

- Remove `effort-2025-11-24` and
  `interleaved-thinking-2025-05-14`; effort is GA and adaptive thinking
  interleaves automatically.
- Remove `token-efficient-tools-2025-02-19` and
  `output-128k-2025-02-19` for Claude 4 and later; they have no effect.
- Do not use `context-1m-2025-08-07` for Sonnet 4 or 4.5. Oversized requests
  now fail instead of gaining a larger window.
- Move workloads off deprecated or retired models and off the legacy
  Workbench and experimental prompt-tool endpoints before their shutdowns.

### Update older tool contracts

- Direct Claude 3.x migrations to Opus 5 or Sonnet 5 must use
  `text_editor_20250728` with `str_replace_based_edit_tool` and
  `code_execution_20260521`.
- A Haiku 3.x to Haiku 4.5 migration instead uses
  `text_editor_20250728` and `code_execution_20250825`.
- Remove `undo_edit`.
- Parse tool arguments with a JSON parser and preserve trailing newlines in
  string arguments.

## High-value request patterns

### Discover capabilities instead of hard-coding

Query `GET /v1/models` or `GET /v1/models/{model_id}` and use
`max_input_tokens`, `max_tokens`, and `capabilities`. Dateless IDs beginning
with the 4.6 generation are fixed snapshots, not evergreen aliases.

Hosted IDs differ:

```text
Claude API / Google Cloud: claude-sonnet-4-6
Amazon Bedrock:            anthropic.claude-sonnet-4-6
Bedrock Opus 4.6:          anthropic.claude-opus-4-6-v1
```

An ID pins weights and configuration, but routing, safety classifiers,
sampling logic, and other serving infrastructure may still change.

### Produce structured JSON

Use the GA `output_config.format` contract and decode the returned text block.
Check `stop_reason` before parsing because refusals and truncation can violate
the requested shape.

```python
response = client.messages.create(
    model=model_id,
    max_tokens=256,
    messages=[{"role": "user", "content": "Extract the order number."}],
    output_config={"format": {
        "type": "json_schema",
        "schema": {
            "type": "object",
            "properties": {"order_number": {"type": "string"}},
            "required": ["order_number"],
            "additionalProperties": False,
        },
    }},
)
data = json.loads(next(b.text for b in response.content if b.type == "text"))
```

Set `strict: true` independently on every tool whose name and input must be
grammar-constrained. Final-output schemas do not constrain tool arguments,
tool results, or thinking.

### Stream tool input safely

Enable unbuffered input per user-defined tool:

```python
tools=[{
    "name": "make_file",
    "eager_input_streaming": True,
    "input_schema": {
        "type": "object",
        "properties": {"text": {"type": "string"}},
    },
}]
```

Accumulate `input_json_delta.partial_json` by content-block index, then parse
only at `content_block_stop`. Eager input is not server-validated and may be
truncated at `max_tokens`; never execute invalid JSON. Return a JSON-library
serialized error tool result that preserves the raw input under
`INVALID_JSON`.

### Aggregate long streams

Use streaming to keep large-output requests connected even when only the
final message is needed. Use `get_final_message()` in Python,
`finalMessage()` in TypeScript, `message.Accumulate(event)` in Go,
`MessageAccumulator` in Java, `Aggregate()` or `CollectAsync()` in C#,
or `accumulated_message` in Ruby. PHP requires manual accumulation.

For interrupted 4.6-and-later output, send the captured most-recent text block
back in a new user continuation message. Do not use assistant prefilling, and
do not attempt to resume partial tool-use or thinking blocks.

## Prompt-cache checklist

- A request-level `cache_control` uses one of four breakpoint slots.
- Put one-hour breakpoints before five-minute breakpoints.
- Keep thinking mode, manual thinking budget, effort, tool definitions, and
  serialized prompt prefixes stable when expecting a hit.
- Inspect all three counters:
  `cache_read_input_tokens`, `cache_creation_input_tokens`, and
  `input_tokens`.
- For a miss, use the cache-diagnosis beta with
  `diagnostics.previous_message_id` and inspect `cache_miss_reason`.
- Pre-warm shared system prompts or tools with an explicit breakpoint,
  `max_tokens: 0`, and a non-whitespace placeholder user message.
- Do not stream a zero-output warm-up or combine it with enabled thinking,
  structured output, forced/`any` tool choice, or Message Batches.
- Wait until the writer response begins before sending parallel cache readers.

## Rate-limit checklist

- Treat RPM, ITPM, and OTPM as independent continuously replenished buckets.
- Ramp traffic gradually; acceleration throttling can return 429 below the
  apparent steady-state limit.
- Obey `retry-after`; bucket reset headers are full-replenishment RFC 3339
  timestamps, not necessarily the earliest retry time.
- For most models, ITPM is `input_tokens + cache_creation_input_tokens`;
  cache reads are excluded. Haiku 3.5 is the exception.
- OTPM is charged as output is generated; requested `max_tokens` does not
  reserve output capacity.
- Account for shared Opus 4.x and Sonnet 4.x pools, workspace safeguards,
  separate fast-mode capacity, and separate Message Batches and Managed
  Agents pools.

## Response handling

- Treat HTTP 200 with `stop_reason: "refusal"` as a refusal, not success.
- Read `stop_details.category`; discard partial output after a mid-stream
  refusal.
- Accept a `fallback` stream content block with start and stop events but no
  deltas.
- With thinking display omitted, accept a thinking block containing only one
  `signature_delta`.
- Inspect `usage.speed`; a requested fast mode may error or silently run at
  standard speed depending on the target.
- Compare schema `enum` and `const` output case-insensitively and avoid values
  distinguished only by capitalization.

## Before shipping

1. Confirm the exact model ID and hosted API surface.
2. Query capabilities and limits rather than inheriting older constants.
3. Remove unsupported prefills, sampling fields, headers, and tool versions.
4. Exercise refusal, truncation, context exhaustion, fallback, and 429 paths.
5. Verify cache behavior from usage counters and cache-miss diagnosis.
6. Audit usage by API key and model before a retirement deadline.
7. Keep sensitive data out of schema grammar elements because compiled
   grammars have different retention characteristics from message content.
