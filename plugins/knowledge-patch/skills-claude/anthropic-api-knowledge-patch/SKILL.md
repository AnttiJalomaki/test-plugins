---
name: anthropic-api-knowledge-patch
description: Anthropic API
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Anthropic API Knowledge Patch

Use this skill for Anthropic Messages integrations, current model IDs, Claude 5
migrations, streamed tools, structured output, prompt caching, model
retirements, and Managed Agents.

Treat model, platform, and beta boundaries as explicit contracts. Prefer
capability discovery and response metadata over cross-target assumptions.

## Reference index

| Reference | Topics |
|---|---|
| [Models and migrations](references/models-and-migrations.md) | IDs, thinking, access, fallback, token budgets, tool migration, image accounting |
| [Tools, betas, and streaming](references/tools-and-streaming.md) | Eager tool input, beta headers, event aggregation, fallback blocks, stream recovery |
| [Structured outputs](references/structured-outputs.md) | JSON schemas, SDK parsing, strict tools, complexity and compliance edges |
| [Prompt caching and rate limits](references/caching-and-rate-limits.md) | Breakpoints, TTLs, invalidation, pre-warming, throttles, pools, spend caps |
| [Platforms and lifecycle](references/platforms-and-lifecycle.md) | Retirements, context limits, WIF, AWS surfaces, tunnels, key expiry |
| [Managed Agents](references/managed-agents.md) | Sessions, overrides, streams, memory, vaults, schedules, webhooks |

## Breaking-change triage

Before changing a production integration, check these high-impact boundaries:

- Claude 5 rejects manual thinking budgets, assistant-message prefills, and
  non-default sampling parameters.
- Opus 4.7 and later and Mythos Preview reject non-default `temperature`,
  `top_p`, and `top_k`, even though SDK types may still expose them.
- Raw message requests use `output_config.format`, not deprecated top-level
  `output_format`; the Python parse helper remains an exception.
- Claude 4.6 and later cannot resume interrupted text through assistant
  prefilling. Send the captured text in a new user message.
- Context exhaustion on Claude 4.5 and later is
  `model_context_window_exceeded`, distinct from `max_tokens`.
- `speed: "fast"` errors on Opus 4.7 and is silently ignored by Opus 4.6.
- The old 1M-context beta does not expand Sonnet 4 or 4.5; oversized requests
  fail.
- Memory API calls must replace the core Managed Agents beta with the current
  memory beta; sending both is an error.
- Legacy Workbench and its experimental prompt endpoints have a scheduled
  shutdown. Export prompts, variables, and evals first.

## Select and validate a model ID

For 4.6 and later, use canonical dateless snapshot IDs:

```text
claude-sonnet-4-6
claude-sonnet-5
```

These IDs are pinned releases, not evergreen aliases. Each receives its own
deprecation and retirement schedule. A pinned snapshot stabilizes weights and
configuration, but serving infrastructure can still change.

Hosted IDs differ:

```text
Claude API / Google Cloud: claude-sonnet-4-6
Amazon Bedrock:            anthropic.claude-sonnet-4-6
Bedrock Opus 4.6:          anthropic.claude-opus-4-6-v1
```

Use `GET /v1/models` or `GET /v1/models/{model_id}` to discover
`max_input_tokens`, `max_tokens`, and `capabilities`. Audit actual callers with
the Console's usage export before retiring an ID.

## Migrate generation controls

Claude 5 defaults to adaptive thinking. A typical request uses:

```python
thinking={"type": "adaptive", "display": "summarized"}
output_config={"effort": "high"}
```

Apply target-specific constraints:

- Fable 5 and Mythos 5 cannot disable thinking.
- Opus 5 can disable it only at `high`, `medium`, or `low`; `xhigh` and `max`
  return HTTP 400, and disabling can expose tool syntax in visible output.
- Sonnet 5 can disable thinking at every effort.
- `max_tokens` is a hard combined ceiling for thinking and visible output.
- Pass thinking blocks back unchanged only to the model that produced them.
  Strip `thinking` and `redacted_thinking` when switching models.

Replace formatting prefills with system instructions or structured output.
Omit non-default sampling parameters and steer with effort. Retokenize real
prompts because Claude 5 model families do not preserve older token counts.

For Opus 5 agentic loops, task budgets provide an advisory allowance across
thinking, tools, results, and output. They do not override `max_tokens`:

```python
betas=["task-budgets-2026-03-13"]
output_config={
    "effort": "high", "task_budget": {"type": "tokens", "total": 128000},
}
```

See [Models and migrations](references/models-and-migrations.md) before moving
tools from Claude 3.x, handling refusals, enabling fallback, or changing
mid-conversation instructions.

## Stream tool input safely

Enable unbuffered input per user-defined tool:

```python
tools=[{
    "name": "make_file",
    "eager_input_streaming": True,
    "input_schema": {"type": "object",
                     "properties": {"text": {"type": "string"}}},
}]
```

Treat streamed JSON as untrusted fragments:

1. Start from the `input: {}` placeholder.
2. Concatenate `input_json_delta.partial_json` by content-block index.
3. Parse only at `content_block_stop`.
4. Reject truncated or invalid JSON; never execute it.
5. Return an error tool result containing the raw input in a JSON-library-
   serialized `INVALID_JSON` wrapper.

Omitting `eager_input_streaming` preserves buffered, server-validated tool
input. The legacy fine-grained beta affects only tools where the field is
unset; explicit `false` remains buffered.

## Constrain outputs and tool arguments

For raw JSON output, set `output_config.format` and decode the returned text
block:

```python
output_config={"format": {
    "type": "json_schema",
    "schema": {"type": "object",
               "properties": {"answer": {"type": "string"}},
               "required": ["answer"], "additionalProperties": False},
}}
```

Set `strict: true` separately on each tool whose arguments need grammar
constraints. A final output schema does not constrain tool calls, tool results,
or thinking.

Always inspect `stop_reason` before decoding. Refusals can fall outside the
schema and `max_tokens` can truncate valid JSON. Do not combine citations or
assistant prefilling with JSON output.

Schema compilation has combined request ceilings and can time out. SDK-derived
schemas may be simplified before reaching the server, then validated against
the richer original schema locally. Read
[Structured outputs](references/structured-outputs.md) before using complex
unions, many optional fields, or sensitive schema literals.

## Consume event streams defensively

Use each SDK's final-message accumulator when streaming only to keep a large
request alive. Event consumers must also accept:

- a server-side `fallback` block that starts and stops without deltas;
- an omitted-thinking block that contains only a `signature_delta`; and
- interruption recovery limited to the most recent partial text block.

Raw HTTP puts multiple beta names in one comma-separated `anthropic-beta`
header. SDKs use a `betas` list. The `ant` CLI accepts one comma-separated
`--beta`; repeated flags currently do not compose.

## Design prompt caching deliberately

A request-level `cache_control` creates an automatic breakpoint at the last
eligible block and consumes one of four slots. It can coexist with explicit
breakpoints, but conflicting TTLs or an exhausted slot count return HTTP 400.

Use these rules to preserve hits:

- Keep replayed tool JSON serialization byte-stable.
- Put every one-hour breakpoint before all five-minute breakpoints.
- Keep thinking mode, manual budget, and effort stable.
- Expect tool-definition changes to invalidate tool, system, and message
  caches.
- Wait until the first response begins before issuing parallel requests that
  depend on its newly written entry.

For pre-warming, use `max_tokens: 0`, an explicit system or tool breakpoint,
and a non-whitespace user placeholder. Match production thinking and effort.
Do not combine zero-output warming with streaming, enabled manual thinking,
structured output, forced tool choice, or Message Batches.

Cache floors, hosted-platform isolation, usage accounting, and precise
invalidation rules are in
[Prompt caching and rate limits](references/caching-and-rate-limits.md).

## Handle throttling and capacity

Messages enforces RPM, ITPM, and OTPM independently with continuously refilled
buckets. Ramp traffic to avoid acceleration limits and honor `retry-after` on
HTTP 429. Do not treat requested `max_tokens` as reserved OTPM capacity.

Capacity pools are not uniformly per model: some Opus 4.x and Sonnet 4.x
releases share family pools, fast mode has a dedicated pool, Message Batches
have a separate queue, and Managed Agents have their own organization-level
limits. Workspace caps can be lower than organization limits.

Use the detailed accounting formulas and response-header meanings in
[Prompt caching and rate limits](references/caching-and-rate-limits.md) before
sizing concurrency or retry policy.

## Respect platform boundaries

Claude Platform on AWS and Amazon Bedrock are separate surfaces with different
infrastructure and feature support. Automatic prompt caching, server-side
fallback, and some API products are not portable across every hosted platform.

Prefer Workload Identity Federation and short-lived credentials where
available. When moving tunnel management, use the Claude API route, required
beta, and WIF scope together. Track API-key expiration and pre-expiration
notifications operationally.

Consult [Platforms and lifecycle](references/platforms-and-lifecycle.md) for
retirement dates, context and output cutovers, AWS surface details, inference
location, tunnel migration, and Enterprise user-management requirements.

## Build Managed Agents safely

The core Managed Agents beta covers API-managed agents, containers, tools,
sessions, and streams. Use session-scoped overrides when a model, prompt, tool,
MCP server, or skill should change without mutating the stored agent.

Place effort inside the agent model configuration. Include `version` on updates
when optimistic concurrency matters; a mismatch returns HTTP 409.

Memory endpoints have a separate beta-header cutover and stricter listing
semantics. Vault injection happens at egress, tool outputs above 100,000
characters spill to a sandbox file, and webhook thread events include their
session-thread identifier. Read [Managed Agents](references/managed-agents.md)
before implementing these surfaces.
