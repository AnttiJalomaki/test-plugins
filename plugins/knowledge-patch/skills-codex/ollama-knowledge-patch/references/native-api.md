# Native REST API

The native server is available at `http://localhost:11434`.

## Control thinking

For thinking models, `/api/generate` and `/api/chat` accept `think` as:

- a boolean; or
- `"low"`, `"medium"`, `"high"`, or `"max"`.

Chat responses keep the thinking process separate in `message.thinking`.

```sh
curl http://localhost:11434/api/chat -d \
  '{"model":"gpt-oss:20b","messages":[{"role":"user","content":"Solve this carefully."}],"think":"high","stream":false}'
```

## Preserve tool-result identity

Tool calls may be streamed. After executing a function, append its result to
chat history as a `tool` message and include `tool_name` so the model can
associate the result with the original function:

```json
{"role":"tool","content":"11 degrees celsius","tool_name":"get_weather"}
```

## Create models from blobs

`POST /api/create` can build from GGUF or Safetensors files uploaded to
`POST /api/blobs/sha256:<digest>`.

- Put each filename-to-digest mapping in `files`.
- Put LoRA adapter mappings in `adapters`.
- Use `HEAD /api/blobs/sha256:<digest>` to avoid uploading an existing blob.

```sh
digest=$(sha256sum model.gguf | cut -d ' ' -f 1)
curl -T model.gguf -X POST \
  "http://localhost:11434/api/blobs/sha256:$digest"
curl http://localhost:11434/api/create \
  -d "{\"model\":\"my-model\",\"files\":{\"model.gguf\":\"sha256:$digest\"}}"
```

For an unquantized source, `POST /api/create` also accepts `quantize`.
Supported values are `q4_K_M`, `q4_K_S`, and `q8_0`; prefer `q4_K_M` or
`q8_0`.

```sh
curl http://localhost:11434/api/create \
  -d '{"model":"llama3.2:quantized","from":"llama3.2:3b-instruct-fp16","quantize":"q4_K_M"}'
```

## Create embeddings

`POST /api/embed` supersedes `/api/embeddings`.

- `input` accepts one string or a list of strings.
- The response contains an embeddings matrix.
- `dimensions` can select the output dimensionality.
- Input truncation defaults to true.
- `truncate: false` makes overlong input fail instead of truncating it.

```sh
curl http://localhost:11434/api/embed -d \
  '{"model":"all-minilm","input":["first text","second text"],"truncate":false}'
```

See [image-generation.md](image-generation.md) for CLI and compatibility API
behavior as well as the native `/api/generate` contract.
