# Tools, Streaming, and Beta Features

## Eager tool-input streaming

Batch attribution: `fine-grained-tool-streaming`.

All models support unbuffered tool-input streaming. Make a streaming request
and enable it independently on a user-defined tool with
`eager_input_streaming: true`. Omitting the field retains buffered,
server-validated parameter streaming.

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

This field replaces the legacy
`fine-grained-tool-streaming-2025-05-14` beta header. The old header enables
eager streaming only for tools where the field is unset; an explicit `false`
still forces buffered streaming.

## Accumulating tool input

A streamed `tool_use` block begins with an `input: {}` placeholder. Concatenate
every `input_json_delta.partial_json` string by content-block index, and parse
only at `content_block_stop`.

Eager fragments are not server-validated and a `max_tokens` stop can truncate
them. Guard the parse and never execute invalid input. Return the failure as an
error tool result whose string content preserves the raw input in an
`INVALID_JSON` wrapper. Use a JSON serializer, not string concatenation.

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_...",
  "is_error": true,
  "content": "{\"INVALID_JSON\":\"<unparseable input>\"}"
}
```

## Supplying multiple betas

Raw HTTP puts beta names in one comma-separated `anthropic-beta` header. SDK
beta calls use a `betas` list. The `ant` CLI likewise needs one comma-separated
`--beta` value: if the flag is repeated, only the first currently takes
effect.

```text
anthropic-beta: feature1,feature2
--beta feature1,feature2
```

## Aggregating a complete streamed message

For large `max_tokens`, stream to keep the connection active even if
incremental output is not needed, then obtain the same complete `Message` a
non-streaming request would return.

```python
with client.messages.stream(
    model=model_id,
    max_tokens=128000,
    messages=messages,
) as stream:
    message = stream.get_final_message()
```

SDK aggregation helpers are:

- Python: `get_final_message()`
- TypeScript: `finalMessage()`
- Go: `message.Accumulate(event)`
- Java: `MessageAccumulator`
- C#: `Aggregate()` or `CollectAsync()`
- Ruby: `accumulated_message`
- PHP: manual event accumulation

## Unusual event sequences

At every server-side model fallback boundary, the stream emits a `fallback`
content block as a `content_block_start` and `content_block_stop` pair with no
deltas between them. Event consumers must accept this empty lifecycle.

With thinking configured as `display: "omitted"`, no `thinking_delta` events
arrive. A thinking block still opens, receives one `signature_delta`, and
closes:

```text
content_block_start(thinking) → signature_delta → content_block_stop
```

## Recovering an interrupted stream

Claude 4.5 and earlier can resume by prefilling a new assistant message with
captured output. Claude 4.6 and later instead require a new user message that
contains the output and asks the model to continue.

```python
messages.append({
    "role": "user",
    "content": (
        f"Your previous response was interrupted and ended with "
        f"{partial_text}. Continue from where you left off."
    ),
})
```

Only the most recent text block can be resumed. Partially streamed tool-use
and thinking blocks cannot be recovered.

## Advisor and hosted-tool response pruning

The beta advisor tool pairs a faster executor with a higher-intelligence
advisor during generation and requires:

```text
Anthropic-Beta: advisor-tool-2026-03-01
```

Set `tools[].max_tokens` to cap each advisor response when full-length advice
is unnecessary.

The `web_search_20260318` and `web_fetch_20260318` hosted tools add
`response_inclusion`, allowing consumed result blocks to be left out of the
API response in agentic loops.
