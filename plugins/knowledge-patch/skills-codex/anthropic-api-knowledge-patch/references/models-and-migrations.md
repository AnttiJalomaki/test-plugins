# Models, Migrations, and Lifecycle

## Model identifiers

Batch attribution: `claude-4.6-model-ids`.

Starting with the 4.6 generation, canonical IDs have the form
`claude-{name}-{major}[-{minor}]`; a whole-major release omits the minor.
Unlike short dateless aliases for older releases, these dateless IDs are
pinned snapshots. An update receives a new ID, and each ID receives its own
deprecation and retirement schedule.

```text
claude-sonnet-4-6
claude-sonnet-5
```

Amazon Bedrock uses `anthropic.claude-{name}-{major}[-{minor}]`. Opus 4.6 is
the final exception retaining `-v1`; Sonnet 4.6 and later omit it. Google
Cloud uses the same dateless ID as the Claude API for 4.6 and later.

```text
anthropic.claude-opus-4-6-v1
anthropic.claude-sonnet-4-6
claude-sonnet-4-6
```

A pinned ID fixes model weights and configuration. Routing, safety
classifiers, sampling logic, and other serving infrastructure may still
change, so a minor observable change does not prove that the snapshot changed.

## Claude 5 request migration

Batch attribution: `claude-5-migration`.

### Thinking and effort

All Claude 5 targets default to adaptive thinking and reject manual extended
thinking with `budget_tokens`. Fable 5 and Mythos 5 cannot disable thinking.
Opus 5 permits `thinking: {"type": "disabled"}` only at `high`, `medium`, or
`low` effort and returns HTTP 400 at `xhigh` or `max`. Sonnet 5 permits
disabled thinking at every effort level.

Readable thinking is absent unless `display: "summarized"` is requested.
`max_tokens` remains the hard ceiling over thinking and visible output.
Disabling thinking on Opus 5 can expose tool calls as text or leak internal
XML into visible output.

```python
thinking={"type": "adaptive", "display": "summarized"},
output_config={"effort": "high"},
```

Pass `thinking` blocks back unchanged when continuing on the model that
created them. Before replaying history on a different model, remove both
`thinking` and `redacted_thinking`; foreign-model blocks are silently ignored
but add unnecessary payload.

Opus 5 supports the beta task-budget facility for an advisory running token
allowance across thinking, tool calls, tool results, and final output. The
minimum is 20,000 tokens. It does not replace the hard per-request
`max_tokens` ceiling.

```python
betas=["task-budgets-2026-03-13"],
output_config={
    "effort": "high",
    "task_budget": {"type": "tokens", "total": 128000},
}
```

At `xhigh` or `max` effort, begin Opus 5 workloads with at least 64,000
`max_tokens` so thinking and tools have room.

### Prefill, sampling, and output format

Fable 5, Mythos 5, Opus 5, and Sonnet 5 reject assistant-message prefills.
Non-default `temperature`, `top_p`, and `top_k` return HTTP 400. Replace
formatting prefills with system instructions or structured output. Move the
deprecated raw request field `output_format` to `output_config.format`.

```python
output_config={"format": output_format}
```

The SDK types may still expose the sampling fields, but the same non-default
sampling rejection applies to Opus 4.7 and later and Mythos Preview. Use
effort to steer newer targets.

### Access, retention, refusals, and fallback

Fable 5 is generally available, supports Priority Tier, and runs safety
classifiers. Mythos 5 is restricted to approved Project Glasswing customers,
does not support Priority Tier, and omits those classifiers. Both require
30-day retention and are unavailable with zero-data retention. An ineligible
Fable request on the Claude API returns HTTP 400 `invalid_request_error`;
retention can instead be configured per workspace.

Fable 5 and Opus 5 classifier refusals are HTTP 200 responses with
`stop_reason: "refusal"` and `stop_details.category`. Fable categories include
`cyber`, `bio`, and `reasoning_extraction`. A Fable pre-output refusal is not
billed for input tokens. Discard partial output from a mid-stream refusal.

Fable 5 accepts the beta `fallbacks` parameter. Opus 5 can ask the server to
choose a recommended classifier-dependent target using
`fallbacks: "default"` and:

```text
Anthropic-Beta: server-side-fallback-2026-07-01
{"fallbacks":"default"}
```

Server-side fallback is unavailable in Message Batches and hosted-platform
APIs.

### Tokenizers and context

Sonnet 5 uses about 30% more tokens than Sonnet 4.6 while retaining a default
one-million-token context and a 128,000-token maximum output. Opus 5 uses the
tokenizer introduced in Opus 4.7, which may use roughly 1 to 1.35 times the
tokens of pre-4.7 models; its one-million-token context needs no beta header
and has no long-context premium.

Fable 5 and Mythos 5 share the Mythos Preview tokenizer and permit up to
128,000 output tokens. Rerun token counting and retune `max_tokens` and
compaction thresholds instead of preserving older counts.

Claude 4.5 and later report context exhaustion as
`stop_reason: "model_context_window_exceeded"`, distinct from the requested
output ceiling's `max_tokens`.

### Instructions, tools, and cache boundary

Opus 5 accepts a `role: "system"` message in `messages` immediately after a
user turn. Initial instructions remain in top-level `system`. Sonnet 5 does
not support mid-conversation system messages.

The `mid-conversation-tool-changes-2026-07-01` beta lets Opus 5 add or remove
tools between turns without invalidating cache hits for earlier turns.
Opus 5, Fable 5, and Mythos 5 reduce the minimum cacheable prompt to 512
tokens.

```text
messages=[
  {"role":"user","content":"Apply the new policy."},
  {"role":"system","content":"Use the revised instructions."}
]
Anthropic-Beta: mid-conversation-tool-changes-2026-07-01
```

When jumping directly from Claude 3.x to Opus 5 or Sonnet 5, use
`text_editor_20250728` with `str_replace_based_edit_tool` and
`code_execution_20260521`; remove `undo_edit`. Use a standard JSON parser for
tool inputs because escaping can differ. Preserve the trailing newlines that
Claude 4.5 and later retain in string arguments.

Remove the obsolete `effort-2025-11-24` and
`interleaved-thinking-2025-05-14` beta headers. Also remove
`token-efficient-tools-2025-02-19` and `output-128k-2025-02-19`, which have no
effect on Claude 4 and later.

### Opus 5 capabilities and prompt retuning

Opus 5 offers research-preview fast mode with `speed: "fast"` and:

```text
Anthropic-Beta: fast-mode-2026-02-01
{"speed":"fast"}
```

It does not support web fetch or Priority Tier.

Opus 5 visible responses tend to be longer even when lower effort reduces
thinking. State the desired deliverable length. Remove inherited self-check
instructions that cause over-verification, keep narrow tasks constrained, and
state when delegation is warranted or cap subagent counts because Opus 5
delegates more readily.

### Image accounting

Opus 4.7 and later automatically accept images up to 2,576 pixels on the long
edge or 3.75 megapixels. A full-resolution image can cost about 4,784 tokens,
roughly three times the older cap. Returned pointing and bounding-box
coordinates correspond 1:1 to image pixels; remove old scale-factor
conversion. Downsample when the extra fidelity is unnecessary.

## Haiku 4.5 boundary

Haiku 4.5 retains optional manual extended thinking and rejects adaptive
thinking. When moving from Haiku 3.x, send only one of `temperature` or
`top_p`, update to `text_editor_20250728` and `code_execution_20250825`, and
remove `undo_edit`. Haiku 4.5 has rate limits separate from Haiku 3.5.

## Lifecycle and retirement operations

Legacy models receive no updates but do not yet have retirement dates.
Deprecated models remain callable until their retirement; retired-model
requests fail. Public releases receive at least 60 days' retirement notice.
Use the Console usage export, grouped by API key and model, to locate callers.

The current schedule in this guidance is:

- `claude-mythos-preview` is deprecated in favor of `claude-mythos-5`.
- Move `claude-opus-4-1-20250805` to `claude-opus-4-8` before August 5,
  2026.
- Claude 4.0, 3.x, 2.x, 1.x, and Instant IDs listed as retired are no longer
  callable.
- Earliest tentative dates are November 24, 2026 for Opus 4.5; September 29,
  2026 for Sonnet 4.5; and October 15, 2026 for Haiku 4.5.
- Earliest tentative dates are February 5, April 16, May 28, and July 24,
  2027 for Opus 4.6, 4.7, 4.8, and 5 respectively.
- Earliest tentative dates are February 17 and June 30, 2027 for Sonnet 4.6
  and 5, and June 9, 2027 for Fable 5.

Legacy Workbench access ends August 17, 2026. Its prompts, variables, and evals
do not transfer to the updated experience, so export them first. The
experimental `/v1/experimental/generate_prompt`,
`/v1/experimental/improve_prompt`, and
`/v1/experimental/templatize_prompt` endpoints retire the same day and then
return errors.

## Context and fast-mode cutovers

The `context-1m-2025-08-07` header has no effect on Sonnet 4 or 4.5; those
models reject requests beyond their standard 200,000-token context. Opus 4.6
and Sonnet 4.6 instead provide one million tokens at standard pricing without
a beta header, use ordinary account rate limits at every context length, and
accept up to 600 images or PDF pages in a one-million-token request.

`speed: "fast"` on Opus 4.7 returns an error. On Opus 4.6, it silently uses
standard speed and pricing. Inspect `usage.speed` and move fast workloads to a
supported target.
