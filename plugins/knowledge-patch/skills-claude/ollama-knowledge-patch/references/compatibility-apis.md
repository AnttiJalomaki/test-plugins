# Compatibility APIs

## Anthropic Messages clients

Ollama 0.14.0 and later accepts Anthropic Messages clients at the server root.
Clients require an API key or authentication token, but Ollama ignores its
value. Supported features include multi-turn messages, streaming, system
prompts, tool calling, extended thinking, and image input.

```sh
export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_BASE_URL=http://localhost:11434
claude --model gpt-oss:20b
```

## OpenAI client setup

Point clients at `http://localhost:11434/v1/`. A required API key can contain
any value because Ollama ignores it.

## Chat Completions

`/v1/chat/completions` supports streaming usage, JSON mode, seeded output,
tools, and vision. Image content must be base64 encoded; remote image URLs are
not supported.

Unsupported fields are `tool_choice`, log probabilities, `logit_bias`, `user`,
and `n`.

Thinking models accept either `reasoning_effort` or `reasoning.effort`. Valid
efforts are `"high"`, `"medium"`, `"low"`, `"max"`, and `"none"`.

```json
{
  "model": "gpt-oss:20b",
  "messages": [{"role": "user", "content": "Answer briefly."}],
  "reasoning_effort": "none"
}
```

## Completions

`/v1/completions` requires `prompt` to be a string. It supports `suffix` but
does not support `best_of`, `echo`, log probabilities, `logit_bias`, `user`,
or `n`.

## Embeddings

`/v1/embeddings` accepts a string or an array of strings, an encoding-format
selector, and `dimensions`. It does not accept token arrays or `user`.

## Model metadata

For `/v1/models` and `/v1/models/{model}`, `created` is the model's
last-modified time. `owned_by` is the Ollama username and defaults to
`"library"`.

## Stateless Responses

Ollama 0.13.3 adds `/v1/responses` with streaming, function tools, and
reasoning summaries. It accepts `input`, `instructions`, `temperature`,
`top_p`, and `max_output_tokens`.

It does not support `previous_response_id`, `conversation`, or `truncation`.
Applications must keep and resend their own conversation state.

```python
response = client.responses.create(
    model="qwen3:8b",
    input="Write a short poem about blue",
)
print(response.output_text)
```

## Experimental image endpoint

`/v1/images/generations` accepts `model`, `prompt`, and `size`.
`response_format` must be `b64_json`. The endpoint does not support `n`,
`quality`, `style`, or `user`.

```python
response = client.images.generate(
    model="x/z-image-turbo",
    prompt="A robot learning to paint",
    size="1024x1024",
    response_format="b64_json",
)
```

This endpoint is experimental and can change or be removed.
