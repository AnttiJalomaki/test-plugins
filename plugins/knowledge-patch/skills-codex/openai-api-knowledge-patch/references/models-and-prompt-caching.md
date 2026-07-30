# Models, Reasoning, Media, and Prompt Caching

## GPT-5.6 family selection

The `gpt-5.6` alias routes to flagship `gpt-5.6-sol`. Choose
`gpt-5.6-terra` for a balanced lower-cost tier and `gpt-5.6-luna` for
efficient high-volume work. (`gpt-5.6`; 2026-07-09)

Sol and Terra have roughly 1.05 million input-context tokens; Luna has 400,000.
All three support up to 128,000 output tokens. A Sol or Terra request exceeding
272,000 input tokens enters a different full-request pricing tier.

## Reasoning levels and endpoint fields

The family supports `none`, `low`, `medium`, `high`, `xhigh`, and `max`.
Omission defaults to `medium` in both standard and Pro modes.

- Responses uses `reasoning: {"effort": "none"}`.
- Chat Completions uses `reasoning_effort: "none"`.

When migrating between endpoints, preserve the old effective effort first;
tune it only after verifying behavior.

## Chat Completions function tools

GPT-5.6-family function tools in Chat Completions require effective reasoning
`none`. The omitted default of `medium` is incompatible. Set the field
explicitly or move reasoning-plus-tools work to Responses.

```json
{
  "model": "gpt-5.6-luna",
  "reasoning_effort": "none",
  "tools": [{
    "type": "function",
    "function": {
      "name": "lookup",
      "parameters": {"type": "object", "properties": {}}
    }
  }]
}
```

## Pro reasoning mode

Pro is a Responses-only reasoning mode on an ordinary family model, not a
separate model slug. Mode and effort are independent. Supported Pro efforts
begin at `medium`.

```json
{
  "model": "gpt-5.6-sol",
  "reasoning": {
    "mode": "pro",
    "effort": "medium"
  }
}
```

## Multimodal detail

Omitted image detail, or `auto`, can retain original image dimensions.
Responses file inputs with omitted detail or `input_file.detail: "auto"` can
use high-detail PDF page images. Both behaviors can increase input tokens and
latency.

Chat Completions file inputs do not expose the same detail control. Set detail
explicitly on endpoints and block types that support it whenever cost or
latency needs to be bounded.

## Implicit and explicit caching

Implicit caching places a managed breakpoint near the latest user or tool
message and no longer rounds boundaries to 128 tokens. A changing suffix can
therefore displace an otherwise stable prefix.

Use `prompt_cache_options` with `prompt_cache_breakpoint` when a measured,
stable boundary is needed. `prompt_cache_options.ttl` replaces the deprecated
`prompt_cache_retention` field for GPT-5.6-family explicit caching:

```json
{
  "prompt_cache_options": {
    "mode": "explicit",
    "ttl": "30m"
  }
}
```

Cache writes appear as `cache_write_tokens` and cost 1.25 times the uncached
input-token price.

## Cache keys and sharding

On GPT-5.6 and later families, set `prompt_cache_key` to use the more reliable
matching path for implicit and explicit caching:

```json
{"prompt_cache_key": "tenant:acme:knowledge-base-v1"}
```

A shared key is only a routing aid: it hits only when the exact prefix through
a breakpoint matches. Keep total traffic for all prefixes sharing a key near
15 requests per minute. Shard busier workloads with a stable mapping that
continues to route identical prefixes to the same key.

## Breakpoint write and lookup limits

One request can create at most four cache writes:

- Implicit mode uses one slot for its managed breakpoint and can write the
  latest three explicit breakpoints.
- Explicit mode can write the latest four explicit breakpoints.
- Breakpoints inherited from earlier turns remain read-only.

Lookup examines at most the latest 50 breakpoints and selects the longest
matching prefix. The rendered prefix must contain at least 1,024 tokens.

## Compatible breakpoint blocks

`prompt_cache_breakpoint` terminates the prefix after its annotated block. Its
only valid mode is `explicit`.

Supported Responses blocks are `input_text`, `input_image`, and `input_file`.
Supported Chat Completions blocks are `text`, `image_url`, `input_audio`,
`file`, and `refusal`.

Request-wide explicit mode without any breakpoint marker disables both cache
use and cache writes. Unsupported or non-cacheable blocks return
`400 invalid_request_error`. Models older than GPT-5.6 reject
`prompt_cache_breakpoint` and `prompt_cache_options`; leave those models on
their automatic caching behavior.

## Retention by model generation

For GPT-5.6 and later families, `prompt_cache_options.ttl` is a minimum
lifetime, not a maximum-retention policy. The only accepted value and default
is `30m`; a prefix can remain reusable beyond 30 minutes.

Earlier models use the maximum-policy field `prompt_cache_retention`:

- `in_memory` normally survives 5–10 minutes of inactivity, with a one-hour
  maximum.
- `24h` has a 24-hour maximum.

## Extended retention compatibility

The `24h` policy is supported by:

- `gpt-5.5`, `gpt-5.5-pro`, `gpt-5.4`, `gpt-5.2`
- `gpt-5.1-codex-max`, `gpt-5.1`, `gpt-5.1-codex`,
  `gpt-5.1-codex-mini`, `gpt-5.1-chat-latest`
- `gpt-5`, `gpt-5-codex`, and `gpt-4.1`

The GPT-5.5 pair accepts only `24h`. On older models accepting both policies,
omission defaults to `24h` without Zero Data Retention and to `in_memory` with
Zero Data Retention.
