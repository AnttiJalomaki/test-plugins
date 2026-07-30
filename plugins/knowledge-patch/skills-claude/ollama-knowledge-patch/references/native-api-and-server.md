# Native API and server operation

## Thinking controls

For thinking models, `/api/generate` and `/api/chat` accept `think` as a
boolean or one of `"low"`, `"medium"`, `"high"`, and `"max"`. Chat responses
keep the thinking process separate in `message.thinking`.

```sh
curl http://localhost:11434/api/chat -d \
  '{"model":"gpt-oss:20b","messages":[{"role":"user","content":"Solve this carefully."}],"think":"high","stream":false}'
```

## Tool calls and results

Tool calls can be streamed. After executing a function, append its result to
chat history as a `tool` message and include `tool_name`; this associates the
result with the correct function.

```json
{"role":"tool","content":"11 degrees celsius","tool_name":"get_weather"}
```

## Embeddings

`POST /api/embed` supersedes `/api/embeddings`. It accepts one string or a list
of strings in `input` and returns an embeddings matrix. Use `dimensions` to
control output dimensions.

Truncation defaults to true. Set `truncate: false` when overlong input must
produce an error rather than be shortened.

```sh
curl http://localhost:11434/api/embed -d \
  '{"model":"all-minilm","input":["first text","second text"],"truncate":false}'
```

## Experimental image generation

The standard `/api/generate` endpoint detects image-generation models.
`width`, `height`, and `steps` control output. Streaming responses report
`completed` and `total`; the final `image` field contains base64 data.

```sh
curl http://localhost:11434/api/generate -d \
  '{"model":"x/z-image-turbo","prompt":"a sunset over mountains","width":1024,"height":768}'
```

This API is experimental.

## Disable cloud features

For local-only operation, set `disable_ollama_cloud` in
`~/.ollama/server.json`:

```json
{
  "disable_ollama_cloud": true
}
```

The environment alternative is `OLLAMA_NO_CLOUD=1`. Restart the server after
changing either setting. Cloud models and web search are then disabled; the
logs should report `Ollama cloud disabled: true`.

## Concurrency and queue limits

- `OLLAMA_MAX_LOADED_MODELS` defaults to three times the GPU count, or three
  under CPU inference.
- `OLLAMA_NUM_PARALLEL` defaults to one request per model.
- `OLLAMA_MAX_QUEUE` defaults to 512; excess requests receive HTTP 503.

Parallelism multiplies both context allocation and required memory by the
parallel-request count.

```sh
OLLAMA_MAX_LOADED_MODELS=2 \
OLLAMA_NUM_PARALLEL=4 \
OLLAMA_MAX_QUEUE=128 \
ollama serve
```

## Flash Attention and K/V cache quantization

Flash Attention is selected automatically when supported. Force it on with
`OLLAMA_FLASH_ATTENTION=1` or off with `OLLAMA_FLASH_ATTENTION=0`.

With Flash Attention enabled, `OLLAMA_KV_CACHE_TYPE` changes the global cache
default from `f16`. The `q8_0` cache uses roughly half the memory, while `q4_0`
uses roughly one quarter. More aggressive quantization brings increasing
quality loss.

```sh
OLLAMA_FLASH_ATTENTION=1 OLLAMA_KV_CACHE_TYPE=q8_0 ollama serve
```
