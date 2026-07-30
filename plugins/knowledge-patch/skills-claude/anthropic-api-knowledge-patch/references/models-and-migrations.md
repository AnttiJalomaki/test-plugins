# Models and Migrations

Use this reference when selecting a model ID, moving a conversation between
models, or retuning an integration for Claude 5. It incorporates the
`claude-4.6-model-ids` and `claude-5-migration` compatibility batches.

## Model identifiers and snapshot behavior

Starting with the 4.6 generation, canonical IDs are dateless and use
`claude-{name}-{major}[-{minor}]`. Whole-major releases omit the minor segment,
as in `claude-sonnet-5`. These are fixed snapshots, not the evergreen short
dateless aliases used for earlier releases. An updated release gets a new ID,
and every ID has its own deprecation and retirement schedule.

Hosted platforms use these forms:

- Amazon Bedrock: `anthropic.claude-{name}-{major}[-{minor}]`. Opus 4.6 is the
  final `-v1` exception (`anthropic.claude-opus-4-6-v1`); Sonnet 4.6 and later
  omit the suffix.
- Google Cloud: the same dateless ID as the Claude API for 4.6 and later, such
  as `claude-sonnet-4-6`.

A pinned ID fixes weights and configuration, but not routing, safety
classifiers, sampling logic, or other serving infrastructure. Small observable
changes do not by themselves indicate a changed snapshot.

## Claude 5 thinking and generation controls

All Claude 5 targets default to adaptive thinking and reject manual extended
thinking with `budget_tokens`.

- Fable 5 and Mythos 5 cannot disable thinking.
- Opus 5 accepts `thinking: {"type": "disabled"}` only at `high`, `medium`, or
  `low` effort; it returns HTTP 400 at `xhigh` or `max`. Disabling thinking can
  expose tool calls as text or internal XML in visible output.
- Sonnet 5 can disable thinking at every effort level.
- Readable thinking is absent unless `display: "summarized"` is requested.
- `max_tokens` remains the hard ceiling across thinking and visible output.

```python
thinking={"type": "adaptive", "display": "summarized"}
output_config={"effort": "high"}
```

Fable 5, Mythos 5, Opus 5, and Sonnet 5 reject assistant-message prefills and
non-default `temperature`, `top_p`, or `top_k` with HTTP 400. Replace formatting
prefills with system instructions or structured output, use effort for steering,
and move the deprecated top-level `output_format` to `output_config.format`.

When continuing with the same model, replay `thinking` blocks unchanged. Before
replaying history on a different model, remove both `thinking` and
`redacted_thinking`; foreign-model blocks are silently ignored but waste payload.

## Access, retention, refusals, and fallback

`claude-fable-5` is generally available, supports Priority Tier, and runs safety
classifiers. `claude-mythos-5` is restricted to approved Project Glasswing
customers, does not support Priority Tier, and omits those classifiers. Both
require 30-day retention and are unavailable with zero-data retention. On the
Claude API, an ineligible Fable request returns HTTP 400 `invalid_request_error`;
retention may instead be configured at workspace level.

Fable 5 and Opus 5 classifier refusals are HTTP 200 messages with
`stop_reason: "refusal"` and `stop_details.category`. Fable categories include
`cyber`, `bio`, and `reasoning_extraction`. A pre-output Fable refusal is not
billed for input tokens; discard any partial output from a mid-stream refusal.

Fable 5 supports the beta `fallbacks` parameter. Opus 5 can use
`fallbacks: "default"` to select a recommended classifier-dependent target when
the `server-side-fallback-2026-07-01` beta is enabled. Server-side fallback is
not available to Message Batches or hosted-platform APIs.

## Tokenization, context, and budgets

Retokenize real prompts after migration:

- Sonnet 5 uses about 30% more tokens than Sonnet 4.6, with a default 1M context
  window and 128k maximum output.
- Opus 5 uses the Opus 4.7 tokenizer, commonly consuming about 1–1.35 times the
  tokens of pre-4.7 models. Its 1M context needs no beta header or long-context
  premium.
- Fable 5 and Mythos 5 share the Mythos Preview tokenizer and allow up to 128k
  output.

Retune `max_tokens`, token counting, and compaction thresholds instead of
carrying older counts forward.

Opus 5 beta task budgets expose an advisory running allowance across thinking,
tool calls, tool results, and final output. The minimum is 20k tokens. This does
not replace the hard per-request `max_tokens` ceiling.

```python
betas=["task-budgets-2026-03-13"]
output_config={
    "effort": "high",
    "task_budget": {"type": "tokens", "total": 128000},
}
```

For high-effort Opus 5 work at `xhigh` or `max`, start with at least 64k
`max_tokens` so thinking and tools have room. Opus 5 visible responses also run
longer even when lower effort reduces thinking, so state the desired deliverable
length explicitly.

Claude 4.5 and later report context exhaustion as
`stop_reason: "model_context_window_exceeded"`; handle it separately from the
requested `max_tokens` output ceiling.

## Conversation instructions and cache-friendly changes

Opus 5 accepts a `role: "system"` message in `messages` immediately after a user
turn. Initial instructions remain in the top-level `system` field. Sonnet 5 does
not support mid-conversation system messages.

The `mid-conversation-tool-changes-2026-07-01` beta lets Opus 5 add or remove
tools between turns without invalidating cache hits on earlier turns. Opus 5,
Fable 5, and Mythos 5 also reduce the minimum cacheable prompt to 512 tokens.

## Opus 5 surface and media behavior

Opus 5 research-preview fast mode uses `speed: "fast"` with the
`fast-mode-2026-02-01` beta. Opus 5 does not support web fetch or Priority Tier.

Opus 4.7 and later accept images automatically up to 2,576 pixels on the long
edge or 3.75 megapixels. A full-resolution image can consume about 4,784 tokens,
roughly three times the older cap. Point and bounding-box coordinates are now
1:1 image pixels; remove scale-factor conversion and downsample when the added
fidelity is unnecessary.

## Tool compatibility from older models

Direct migrations from Claude 3.x to Opus 5 or Sonnet 5 must use
`text_editor_20250728` with `str_replace_based_edit_tool` and
`code_execution_20260521`, and remove `undo_edit`. Parse tool inputs with a
standard JSON parser because escaping can differ. Preserve trailing newlines,
which Claude 4.5 and later retain in string arguments.

Haiku 4.5 remains a compatibility boundary: it supports optional manual
extended thinking and rejects adaptive thinking. When moving from Haiku 3.x,
send only one of `temperature` or `top_p`, update to `text_editor_20250728` and
`code_execution_20250825`, and remove `undo_edit`. Haiku 4.5 has separate rate
limits from Haiku 3.5.

Remove retired beta headers when modernizing:

- Effort is GA and adaptive thinking already enables interleaved thinking, so
  remove `effort-2025-11-24` and `interleaved-thinking-2025-05-14`.
- `token-efficient-tools-2025-02-19` and `output-128k-2025-02-19` have no effect
  on Claude 4 and later.

## Prompt and agent-scaffold retuning

Remove inherited self-check instructions that cause excessive verification,
constrain narrow tasks explicitly, and say when delegation is warranted or cap
subagent counts because Opus 5 delegates more readily. Prompt explicitly for a
short deliverable when that is the requirement.
