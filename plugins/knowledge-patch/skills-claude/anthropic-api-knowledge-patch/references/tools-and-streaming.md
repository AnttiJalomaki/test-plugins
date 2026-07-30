# Tools, Betas, and Streaming

Use this reference for tool definitions, streamed event consumers, interrupted
responses, and beta-header composition. The eager-input behavior comes from the
`fine-grained-tool-streaming` compatibility batch.

## Per-tool eager input streaming

All models support unbuffered tool-input streaming. Make a streaming request and
set `eager_input_streaming: true` on each user-defined tool that needs it.
Omitting the field retains buffered, server-validated parameter streaming.

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

The per-tool field replaces `fine-grained-tool-streaming-2025-05-14`. The
legacy beta header enables eager streaming only where the field is unset; an
explicit `false` still forces buffered streaming.

A streamed `tool_use` block begins with `input: {}`. Accumulate every
`input_json_delta.partial_json` string by content-block index and parse only at
`content_block_stop`. Eager fragments are not server-validated and may be
truncated by a `max_tokens` stop, so catch parse failures and never execute
invalid input.

Return invalid input as an error tool result whose string content preserves the
raw input inside an `INVALID_JSON` wrapper. Serialize the wrapper with a JSON
library, never string concatenation.

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_...",
  "is_error": true,
  "content": "{\"INVALID_JSON\":\"<unparseable input>\"}"
}
```

## Multiple beta features

Raw HTTP combines beta names in one comma-separated `anthropic-beta` header.
SDK beta calls use a `betas` list. The `ant` CLI also requires one
comma-separated `--beta` value; repeating the flag currently honors only the
first occurrence.

```text
anthropic-beta: feature1,feature2
ant --beta feature1,feature2
```

## Aggregating a complete streamed message

For large `max_tokens`, stream to keep the connection alive even when the
application only needs the completed message. SDK completion helpers produce
the same `Message` shape as a non-streaming request:

- Python: `get_final_message()`
- TypeScript: `finalMessage()`
- Go: `message.Accumulate(event)`
- Java: `MessageAccumulator`
- C#: `Aggregate()` or `CollectAsync()`
- Ruby: `accumulated_message`
- PHP: manually accumulate events

```python
with client.messages.stream(
    model=model_id,
    max_tokens=128000,
    messages=messages,
) as stream:
    message = stream.get_final_message()
```

## Unusual event-block lifecycles

At each server-side model fallback boundary, the stream emits a `fallback`
content block as `content_block_start` followed by `content_block_stop`, with no
delta between them. Consumers must accept the empty lifecycle.

With thinking configured as `display: "omitted"`, no `thinking_delta` arrives.
A thinking block still opens, receives one `signature_delta`, and closes:

```text
content_block_start(thinking) -> signature_delta -> content_block_stop
```

## Recovering an interrupted stream

Claude 4.5 and earlier can resume by prefilling a new assistant message with
captured partial output. Claude 4.6 and later reject that approach: add a new
user message containing the captured text and an instruction to continue.

```python
messages.append({
    "role": "user",
    "content": (
        "Your previous response was interrupted and ended with "
        f"{partial_text}. Continue from where you left off."
    ),
})
```

Only the most recent text block is resumable. Partially streamed tool-use and
thinking blocks cannot be recovered.

## Advisor and hosted-tool response pruning

The beta advisor tool pairs a faster executor with a higher-intelligence
advisor during generation and requires `advisor-tool-2026-03-01`. Set
`tools[].max_tokens` to cap each advisor response when full-length advice is
unnecessary.

`web_search_20260318` and `web_fetch_20260318` accept `response_inclusion` so
agent loops can omit already-consumed hosted-tool result blocks from the API
response.
