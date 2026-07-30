# Prompt Caching

Use this reference to select implicit or explicit caching, place stable breakpoints, route keys, interpret token accounting, and keep retention fields compatible with model generations.

## Implicit versus explicit caching

Implicit caching places a managed breakpoint near the latest user or tool message. It no longer uses 128-token rounding. A changing suffix can therefore displace the stable prefix an application expected to reuse.

Use explicit caching when a stable boundary has been measured:

```json
{
  "prompt_cache_options": {
    "mode": "explicit",
    "ttl": "30m"
  }
}
```

Annotate the last block of the stable prefix with `prompt_cache_breakpoint`. `prompt_cache_options.ttl` replaces the deprecated `prompt_cache_retention` field on GPT-5.6 and later families.

Cache writes are exposed as `cache_write_tokens` and cost 1.25 times uncached input. Include writes as a separate cost component when measuring whether explicit caching is worthwhile.

## Cache-key routing and sharding

For GPT-5.6 and later, set `prompt_cache_key` to use the more reliable matching path for implicit and explicit caching:

```json
{
  "prompt_cache_key": "tenant:acme:knowledge-base-v1"
}
```

A shared key is only a routing aid. A hit still requires an exact prefix match at a breakpoint.

Keep aggregate traffic across all prefixes that share one key near 15 requests per minute. For busier workloads, shard with a stable mapping that keeps identical prefixes on the same key. A random shard defeats reuse; a single overloaded key weakens routing reliability.

## Write and lookup windows

A request can create no more than four new cache writes:

- Implicit mode uses one slot for its managed breakpoint and can write the latest three explicit breakpoints.
- Explicit mode can write the latest four explicit breakpoints.
- Breakpoints inherited from earlier turns remain read-only and do not consume a new-write slot.

Lookup examines as many as the latest 50 breakpoints and chooses the longest matching prefix. The rendered prefix must contain at least 1,024 tokens to qualify.

## Breakpoint placement and block types

`prompt_cache_breakpoint` ends the reusable prefix after the annotated block. Its only valid mode is `explicit`.

Supported Responses blocks are:

- `input_text`
- `input_image`
- `input_file`

Supported Chat Completions blocks are:

- `text`
- `image_url`
- `input_audio`
- `file`
- `refusal`

Request-wide explicit mode with no breakpoint markers disables caching and cache writes. Unsupported or non-cacheable annotated blocks return `400 invalid_request_error`.

Models before GPT-5.6 reject both `prompt_cache_breakpoint` and `prompt_cache_options`. Leave those models on their automatic caching behavior rather than sending the newer fields.

## Retention on GPT-5.6 and later

`prompt_cache_options.ttl` is a minimum lifetime, not a maximum-retention policy. The only supported value is `30m`, which is also the default. A cached prefix may remain reusable for longer than 30 minutes.

Do not use TTL expiry as an invalidation boundary for sensitive or correctness-critical content; the stated semantics allow retention beyond the minimum.

## Retention on earlier models

Earlier models use the maximum-policy field `prompt_cache_retention`:

- `in_memory` normally survives 5–10 minutes of inactivity and has a one-hour maximum.
- `24h` has a 24-hour maximum.

Extended `24h` retention is supported by:

- `gpt-5.5` and `gpt-5.5-pro`
- `gpt-5.4`
- `gpt-5.2`
- `gpt-5.1-codex-max`
- `gpt-5.1`, `gpt-5.1-codex`, and `gpt-5.1-codex-mini`
- `gpt-5.1-chat-latest`
- `gpt-5` and `gpt-5-codex`
- `gpt-4.1`

The GPT-5.5 pair accepts only `24h`. On older models that accept both policies, omission defaults to `24h` without Zero Data Retention and to `in_memory` with Zero Data Retention.

## Stable-prefix checklist

1. Put invariant instructions and reusable context before changing user/tool content.
2. Measure the rendered prefix and ensure it reaches 1,024 tokens.
3. Select implicit mode for convenience or explicit breakpoints for a controlled boundary.
4. Add a stable `prompt_cache_key`, then shard only when aggregate traffic requires it.
5. Stay within four new writes and place the most valuable breakpoints latest.
6. Track `cache_write_tokens` separately from cached reads and uncached input.
7. Send generation-appropriate fields: `prompt_cache_options` on GPT-5.6+, legacy retention only where supported.
8. Treat TTL as a reuse policy, not an application invalidation guarantee.
