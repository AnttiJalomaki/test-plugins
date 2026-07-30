# Models and Reasoning

This reference covers the `gpt-5.6` family, endpoint-specific reasoning controls, reasoning continuity, Pro mode, multimodal detail, and generation-time safeguards.

## Family selection

The `gpt-5.6` alias routes to the flagship `gpt-5.6-sol` model. Choose among the explicit tiers when cost, latency, or capacity must be controlled:

| Model | Positioning | Approximate input context | Maximum output |
| --- | --- | --- | --- |
| `gpt-5.6-sol` | Flagship | 1.05M | 128K |
| `gpt-5.6-terra` | Balanced lower-cost tier | 1.05M | 128K |
| `gpt-5.6-luna` | Efficient high-volume tier | 400K | 128K |

For Sol and Terra, requests above 272K input tokens use different pricing for the full request, not only the excess portion.

## Reasoning effort by endpoint

The family accepts `none`, `low`, `medium`, `high`, `xhigh`, and `max`. Both standard and Pro modes default to `medium` when effort is omitted.

Responses nests the field:

```json
{
  "model": "gpt-5.6-sol",
  "reasoning": {"effort": "none"}
}
```

Chat Completions uses a top-level field:

```json
{
  "model": "gpt-5.6-luna",
  "reasoning_effort": "none"
}
```

When migrating, first preserve the previous effective effort, then benchmark and tune. This separates endpoint migration regressions from reasoning-level changes.

### Chat Completions tool constraint

Function tools on Chat Completions require effective reasoning `none`. The omitted default, `medium`, is incompatible. Set `reasoning_effort: "none"` explicitly or move a reasoning-plus-tools workflow to Responses.

```json
{
  "model": "gpt-5.6-luna",
  "reasoning_effort": "none",
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "lookup",
        "parameters": {"type": "object", "properties": {}}
      }
    }
  ]
}
```

## Persisted reasoning context

Responses can preserve reasoning context across a `previous_response_id` chain:

```json
{
  "reasoning": {"context": "all_turns"},
  "previous_response_id": "resp_..."
}
```

Choose context by task continuity:

- `all_turns`: retain earlier reasoning only while goals and assumptions remain stable.
- `current_turn`: discard stale earlier reasoning while retaining the current turn's work.
- `auto` or omission: use the service default and inspect the effective value returned in the response.

If the application reconstructs history manually, keep every user input and output Item. Preserve item IDs, call IDs, caller metadata, and assistant phase values; these protocol fields are part of reasoning and tool continuity.

For stateless requests, replay every returned reasoning Item with its default `encrypted_content`. Setting `store: false` alone does not make text-only message replay equivalent to Item replay.

## Pro mode

Pro is a Responses-only reasoning mode applied to a normal family model. It is not a separate model slug. Mode and effort are independent, but the supported effort range in Pro begins at `medium`.

```json
{
  "model": "gpt-5.6-sol",
  "reasoning": {
    "mode": "pro",
    "effort": "medium"
  }
}
```

Do not send Pro mode through Chat Completions or construct names such as `gpt-5.6-pro`.

## Multimodal detail and cost

An omitted image detail or `auto` can preserve original dimensions. In Responses, omitted `input_file.detail` or `input_file.detail: "auto"` can use high-detail images for PDF pages. Both behaviors may increase tokens and latency.

Chat Completions file inputs do not expose the same detail control. Where the endpoint permits, set detail explicitly when cost or latency must be bounded.

## Generation-time safeguards

Cyber and biology safeguards can synchronously review generation. They may refuse output or pause a stream for several seconds, including on legitimate dual-use requests. Streaming clients should tolerate the pause without assuming the connection has failed.

Attach a stable, privacy-preserving `safety_identifier` for each end user on standard requests. For Realtime, use the separate `OpenAI-Safety-Identifier` header described in [realtime.md](realtime.md).

## Validation checklist

1. Confirm the explicit model tier and account for the 272K pricing transition on Sol or Terra.
2. Confirm effort field placement for the chosen endpoint.
3. Set Chat Completions reasoning to `none` before enabling function tools.
4. Choose reasoning context from task continuity, not merely conversation length.
5. Preserve protocol Items and identifiers during manual replay.
6. Set multimodal detail explicitly where variable token use is unacceptable.
7. Test refusal handling and multi-second stream pauses.
