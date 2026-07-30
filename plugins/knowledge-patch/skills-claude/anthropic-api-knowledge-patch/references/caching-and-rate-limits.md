# Prompt Caching and Rate Limits

Use this reference when designing cache breakpoints, diagnosing misses,
pre-warming prompts, sizing concurrency, or handling throttles.

## Automatic caching and breakpoints

A request-level `cache_control` enables automatic caching at the last eligible
block and advances that point as a conversation grows. It can coexist with
block-level controls and consumes one of four breakpoint slots.

- If the final target block already has an explicit control with the same TTL,
  automatic caching is a no-op.
- A different TTL on that target, or four existing explicit breakpoints, makes
  the request fail with HTTP 400.
- If the final block is ineligible, the service walks backward to the nearest
  eligible block. If none exists, it silently skips caching.
- Amazon Bedrock does not support automatic caching.

For a miss, enable `cache-diagnosis-2026-04-07`, send
`diagnostics.previous_message_id`, and inspect returned `cache_miss_reason` for
the first diverging prompt prefix.

## Minimum cacheable prefixes

Prefixes shorter than a model's floor silently run uncached; both
`usage.cache_creation_input_tokens` and `usage.cache_read_input_tokens` stay
zero.

| Minimum | Models |
|---|---|
| 512 tokens | Opus 5, Fable 5, Mythos 5 |
| 1,024 tokens | Opus 4.8, Sonnet 5, Sonnet 4.6, Sonnet 4.5, Opus 4.1, Opus 4 |
| 2,048 tokens | Mythos Preview, Opus 4.7, Haiku 3.5 |
| 4,096 tokens | Opus 4.6, Opus 4.5, Haiku 4.5 |

## TTL ordering and accounting

Five-minute and one-hour entries may coexist only when every one-hour
breakpoint precedes all five-minute breakpoints. Cache writes are split under
`usage.cache_creation`; `ephemeral_5m_input_tokens` plus
`ephemeral_1h_input_tokens` equals `cache_creation_input_tokens`.

```json
{"cache_control": {"type": "ephemeral", "ttl": "1h"}}
```

## Thinking-aware validity

Thinking blocks cannot carry `cache_control`, but replayed thinking in earlier
assistant turns is cached with later content and counts as input on a cache
read. A tool-result-only user turn preserves that cache.

Ordinary user content preserves prior thinking on Opus 4.5+ and Sonnet 4.6+.
Earlier Opus and Sonnet models and every Haiku strip prior thinking and
invalidate later message cache entries.

Changing thinking mode, manual `budget_tokens`, or effort always invalidates
message caches and can also invalidate system or tool caches depending on the
model. Explicitly setting the model's default effort is equivalent to omitting
it and does not invalidate the cache.

## Prefix invalidators and serialization

- Changing tool definitions invalidates tool, system, and message caches.
- Toggling web search or citations, or changing `speed`, preserves only the
  tool cache.
- Changing `tool_choice`, or adding or removing images, preserves tool and
  system caches but invalidates messages.
- Prefix matching is byte-sensitive enough that unstable JSON key ordering in
  replayed `tool_use` blocks can defeat a hit. Use stable serialization.

The mid-conversation tool-change beta is a model-specific exception described
in [Models and Migrations](models-and-migrations.md).

## Zero-output cache pre-warming

Set `max_tokens: 0`, put an explicit breakpoint on shared system text or tools,
and send a non-whitespace placeholder user message. Automatic caching is a poor
fit because it targets the placeholder. Keep thinking and effort identical to
real traffic.

```json
{
  "model": "claude-opus-5",
  "max_tokens": 0,
  "system": [{
    "type": "text",
    "text": "Shared instructions",
    "cache_control": {"type": "ephemeral"}
  }],
  "messages": [{"role": "user", "content": "warmup"}]
}
```

A successful warm-up has `content: []`, `stop_reason: "max_tokens"`, populated
usage, and zero output tokens. Zero-output requests reject `stream: true`,
`thinking.type: "enabled"`, `output_config.format`, forced or `any` tool choice,
and Message Batches.

## Availability, isolation, and replication timing

A newly written cache entry is not available until the first response begins;
parallel followers must wait. Caches are workspace-isolated on the Claude API,
Claude Platform on AWS, and Microsoft Foundry, but organization-isolated on
Bedrock and Google Cloud.

Automatic and explicit caching remain zero-data-retention eligible. Cache
representations and hashes are held only in memory, not stored at rest.

## Continuous throttles and safe retry

The Messages API independently enforces requests per minute, input tokens per
minute, and output tokens per minute using continuously replenished token
buckets. Enforcement can happen over sub-minute intervals. A sharp traffic
increase can trigger an acceleration-limit HTTP 429 even when steady-state
rates appear safe; ramp gradually and obey `retry-after`.

`retry-after` is the number of seconds until a retry can succeed. The response
also includes `anthropic-ratelimit-{requests|tokens|input-tokens|output-tokens}`
families with `limit`, `remaining`, and `reset` variants. Reset values are RFC
3339 timestamps for full replenishment; remaining token values are rounded to
the nearest thousand.

## Spend caps and AWS-billed tiers

Calendar-month caps are $500 on Start, $1,000 on Build, and $200,000 on Scale.
Reaching the cap pauses API use until the next month unless it is raised.
Custom-tier organizations have no standard monthly cap, and any organization
can configure a lower cap.

Claude Platform on AWS organizations begin on Start and do not advance through
usage tiers automatically. AWS Marketplace handles billing, spend limits appear
under Billing instead of Limits, and higher limits require an account
representative or support rather than the ordinary increase-request flow.

## Token accounting

For most models, input-token-per-minute use is:

```text
input_tokens + cache_creation_input_tokens
```

`cache_read_input_tokens` is excluded. Here `input_tokens` covers only content
after the last cache breakpoint; total input is the sum of cache reads, cache
writes, and `input_tokens`. Haiku 3.5 is the exception: cache reads also count
against ITPM.

ITPM is estimated at request start and adjusted to actual input during
processing. OTPM is charged in real time for generated tokens, so requested
`max_tokens` does not reserve output capacity.

## Shared and dedicated pools

Most model limits are independent, with these exceptions:

- Opus 4.5 through 4.8 share one Opus 4.x pool.
- Sonnet 4.5 and 4.6 share one Sonnet 4.x pool.
- Opus 5 and Sonnet 5 each have separate pools.
- `inference_geo: "us"` and `"global"` draw from the same capacity.

Supported `speed: "fast"` requests use a dedicated pool rather than the
standard Opus pool. A fast-pool throttle returns HTTP 429 with `retry-after`;
its state is exposed through `anthropic-fast-*` headers.

Message Batches have a model-independent pool of 1,000 API requests per minute,
at most 200,000 constituent requests awaiting successful processing, and at
most 100,000 constituent requests in one batch. Every constituent item, not
just the enclosing batch, consumes queue capacity.

Managed Agents use a separate organization-level pool: create operations allow
300 requests per minute, while retrieve, list, stream, and other reads allow
1,200 requests per minute.

## Workspace safeguards

A non-default workspace may set lower RPM, ITPM, OTPM, and spend ceilings. An
unset limiter inherits the organization limit, and unused capacity remains
available to other workspaces. The default workspace cannot be capped. The
organization ceiling still applies even if workspace limits sum above it.
