# Compatibility APIs

## Anthropic Messages

Ollama 0.14.0 and later accepts Anthropic Messages clients at the server root.
Clients require an API key or auth token, but Ollama ignores its value.

Supported behavior includes:

- multi-turn messages;
- streaming;
- system prompts;
- tool calling;
- extended thinking; and
- image input.

```sh
export ANTHROPIC_AUTH_TOKEN=ollama
export ANTHROPIC_BASE_URL=http://localhost:11434
claude --model gpt-oss:20b
```

## OpenAI client setup

Point OpenAI clients at:

```text
http://localhost:11434/v1/
```

The client requires an API key, but the local server ignores its value.

## Chat Completions

`/v1/chat/completions` supports:

- streamed usage;
- JSON mode;
- seeded output;
- tools; and
- vision.

Image content must be base64 rather than a remote URL.

Unsupported options are `tool_choice`, log probabilities, `logit_bias`,
`user`, and `n`.

Thinking models accept either `reasoning_effort` or `reasoning.effort`.
Valid efforts are `"high"`, `"medium"`, `"low"`, `"max"`, and `"none"`.

```json
{
  "model": "gpt-oss:20b",
  "messages": [{"role": "user", "content": "Answer briefly."}],
  "reasoning_effort": "none"
}
```

## Legacy Completions

`/v1/completions` accepts only a string `prompt`. It supports `suffix`.

It does not support `best_of`, `echo`, log probabilities, `logit_bias`,
`user`, or `n`.

## Embeddings

`/v1/embeddings` accepts:

- a string or an array of strings;
- an encoding-format selector; and
- `dimensions`.

It does not accept token arrays or `user`.

## Model metadata

For `/v1/models` and `/v1/models/{model}`:

- `created` is the model's last-modified time; and
- `owned_by` is the Ollama username, defaulting to `"library"`.

## Stateless Responses

Ollama 0.13.3 adds `/v1/responses` with streaming, function tools, and
reasoning summaries.

Supported request fields include:

- `input`;
- `instructions`;
- `temperature`;
- `top_p`; and
- `max_output_tokens`.

The endpoint does not support `previous_response_id`, `conversation`, or
`truncation`. Applications must preserve and resend their own conversation
state.

```python
response = client.responses.create(
    model="qwen3:8b",
    input="Write a short poem about blue",
)
print(response.output_text)
```

## Work around hard-coded model names

If a client insists on a default OpenAI model name, copy an existing model to
that name:

```sh
ollama cp llama3.2 gpt-3.5-turbo
```

Use the alias in subsequent API requests.

## Set context outside the request

The compatibility API has no request field for context size. Create and use a
derived model instead:

```text
FROM llama3.2
PARAMETER num_ctx 65536
```

```sh
ollama create mymodel
```

See [image-generation.md](image-generation.md) for the experimental
`/v1/images/generations` contract.
