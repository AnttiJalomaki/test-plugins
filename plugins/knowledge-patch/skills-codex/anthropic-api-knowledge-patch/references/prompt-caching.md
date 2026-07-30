# Prompt Caching

## Automatic caching

A single request-level `cache_control` automatically caches the last cacheable
block and advances the cache point as a conversation grows. It may coexist
with block-level controls and consumes one of the four breakpoint slots.

If the final target block already has an explicit control with the same TTL,
automatic caching is a no-op. A different TTL or four existing explicit
breakpoints returns HTTP 400. If the final block is ineligible, the service
walks backward to the nearest eligible block; if none exists, it silently
skips caching.

Amazon Bedrock does not support automatic caching.

## Model-specific minimum prefixes

Prefixes shorter than the relevant floor silently run uncached, leaving both
`usage.cache_creation_input_tokens` and `usage.cache_read_input_tokens` at
zero.

| Minimum | Models |
| --- | --- |
| 2,048 tokens | Mythos Preview, Opus 4.7, Haiku 3.5 |
| 4,096 tokens | Opus 4.6, Opus 4.5, Haiku 4.5 |
| 1,024 tokens | Opus 4.8, Sonnet 5, Sonnet 4.6, Sonnet 4.5, Opus 4.1, Opus 4 |

Opus 5, Fable 5, and Mythos 5 use a 512-token minimum.

## TTL ordering and accounting

One-hour and five-minute entries may be mixed only when every longer-TTL
breakpoint precedes all shorter-TTL breakpoints.

```json
{"cache_control": {"type": "ephemeral", "ttl": "1h"}}
```

Cache-write usage is split under `usage.cache_creation`.
`ephemeral_5m_input_tokens + ephemeral_1h_input_tokens` equals
`cache_creation_input_tokens`.

## Thinking-aware validity

Thinking blocks cannot carry `cache_control`. Replayed thinking from earlier
assistant turns is cached with later content and counts as input on a read.
A user turn containing only tool results preserves that cache.

Ordinary user content preserves earlier thinking on Opus 4.5 and later and
Sonnet 4.6 and later. Earlier Opus and Sonnet models and every Haiku strip
prior thinking and invalidate subsequent message-cache entries.

Changing thinking mode, manual `budget_tokens`, or effort always invalidates
message caches and may invalidate tool and system caches depending on the
model. Explicitly choosing the model's default effort is equivalent to
omitting it and does not invalidate the cache.

## Prefix invalidators

- Changing tool definitions invalidates tool, system, and message caches.
- Toggling web search or citations, or changing speed, preserves only the
  tool cache.
- Changing `tool_choice`, or adding or removing images, preserves tool and
  system caches but invalidates messages.
- Unstable JSON key ordering in replayed `tool_use` blocks can miss a
  byte-sensitive prefix match.

## Zero-output pre-warming

Warm a shared system prompt or tools by setting `max_tokens: 0`, placing an
explicit breakpoint on the shared block, and sending a non-whitespace
placeholder user message. Automatic caching targets the placeholder, so do
not rely on it for this operation. Keep thinking and effort identical to
production traffic.

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

A successful warm-up returns `content: []`, `stop_reason: "max_tokens"`, a
populated usage object, and zero output tokens.

Zero-output requests reject:

- `stream: true`
- `thinking.type: "enabled"`
- `output_config.format`
- forced or `any` tool choice
- Message Batches

## Availability, isolation, and retention

A new cache entry is unavailable until the first response begins. Parallel
followers must wait for that point.

Caches are workspace-isolated on the Claude API, Claude Platform on AWS, and
Microsoft Foundry. They are organization-isolated on Amazon Bedrock and
Google Cloud.

Automatic and explicit caching remain eligible for zero-data retention. Cache
representations and hashes are held only in memory, not stored at rest.

## Diagnosing misses

Automatic caching may be enabled with a request-level `cache_control`. To
diagnose a miss, send `diagnostics.previous_message_id` while enabling:

```text
Anthropic-Beta: cache-diagnosis-2026-04-07
{"diagnostics":{"previous_message_id":"msg_..."}}
```

Read the returned `cache_miss_reason` to locate the prompt-prefix divergence.
