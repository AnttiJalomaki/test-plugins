---
name: ollama-knowledge-patch
description: Ollama
version: 0.32.5
license: MIT
metadata:
  author: Nevaberry
---


# Ollama Knowledge Patch

Use this skill when working on Ollama commands, Modelfiles, native or
compatibility APIs, integrations, cloud-backed features, image generation, or
runtime sizing. Prefer the project's installed server behavior and model
metadata when they differ from this guidance.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/cli-and-integrations.md](references/cli-and-integrations.md) | CLI entry points, launcher configuration, coding context, model-library tags, and launcher deprecations |
| [references/model-creation.md](references/model-creation.md) | GGUF and Safetensors imports, adapters, blob uploads, quantization, requirements, aliases, and derived context |
| [references/native-api-and-server.md](references/native-api-and-server.md) | Native chat and generate requests, tool results, embeddings, images, cloud controls, concurrency, and cache sizing |
| [references/compatibility-apis.md](references/compatibility-apis.md) | Anthropic Messages and OpenAI-compatible endpoints, supported fields, limitations, and stateless Responses |
| [references/cloud-and-web-tools.md](references/cloud-and-web-tools.md) | Cloud tags, hosted search and fetch, client helpers, coding agents, and MCP exposure |
| [references/acceleration-images-and-scheduling.md](references/acceleration-images-and-scheduling.md) | Vulkan, CUDA, MLX, image CLI controls, accelerator support, and exact memory scheduling |

## Critical compatibility notes

### Skip the withdrawn release

Do not install Ollama 0.32.2. Install or upgrade to 0.32.3 or newer.

### Expect warnings for deprecated launcher tags

The launcher warns before continuing with CodeLlama, Qwen2.5 or
Qwen2.5-coder, Llama 3.x, Mistral, StarCoder, and base DeepSeek-R1 tags. Move
integration configurations to maintained model tags.

### Use the current embeddings endpoint

Use `POST /api/embed` instead of the superseded `/api/embeddings`. It accepts a
single string or a list in `input` and returns an embeddings matrix.

```sh
curl http://localhost:11434/api/embed -d \
  '{"model":"all-minilm","input":["first text","second text"],"truncate":false}'
```

The optional `dimensions` field controls output size. Input truncation defaults
to true; `truncate: false` rejects overlong input.

### Keep Responses conversations in the application

`/v1/responses` is stateless. Do not depend on `previous_response_id`,
`conversation`, or `truncation`; carry prior turns in the application and send
the required context on each request.

### Respect compatibility endpoint limits

The compatibility APIs intentionally implement subsets of their upstream
interfaces. In particular:

- Chat Completions does not support `tool_choice`, log probabilities,
  `logit_bias`, `user`, or `n`.
- Completions requires a string `prompt`; it does not support `best_of`,
  `echo`, log probabilities, `logit_bias`, `user`, or `n`.
- Embeddings does not accept token arrays or `user`.
- Image compatibility responses must use `b64_json`; `n`, `quality`, `style`,
  and `user` are unsupported.

Read [references/compatibility-apis.md](references/compatibility-apis.md) before
porting a client that uses optional request fields.

### Treat image generation as experimental

Native `/api/generate`, the compatibility image endpoint, and the image CLI
workflow are experimental. Check platform support and avoid relying on stable
response or lifecycle guarantees.

## High-value workflows

### Import a local GGUF

Point `FROM` at one GGUF file or a directory containing GGUF files, then create
and run the local model.

```text
FROM ./my-model.Q4_K_M.gguf
```

```sh
ollama create -f Modelfile my-model
ollama run my-model
```

If the imported artifact supports tools, confirm that `ollama show my-model`
lists the `tools` capability before selecting it in an integration.

### Import weights and adapters safely

A supported Safetensors directory can be the Modelfile source:

```text
FROM .
```

Use `ADAPTER` with a Safetensors adapter directory or GGUF adapter file. The
`FROM` model must be the exact base used during fine-tuning; a mismatch can
produce erratic results. Prefer non-quantized Safetensors adapters over QLoRA
adapters because framework quantization methods differ.

### Quantize while creating

Convert an FP16 or FP32 source during creation:

```sh
ollama create --quantize q4_K_M my-model
```

The native create API also accepts `quantize`; supported values are `q4_K_M`,
`q4_K_S`, and `q8_0`, with `q4_K_M` and `q8_0` recommended.

### Control thinking

Native `/api/generate` and `/api/chat` requests accept `think` as a boolean or
as `"low"`, `"medium"`, `"high"`, or `"max"`. Chat returns reasoning separately
in `message.thinking`.

```sh
curl http://localhost:11434/api/chat -d \
  '{"model":"gpt-oss:20b","messages":[{"role":"user","content":"Solve this carefully."}],"think":"high","stream":false}'
```

When returning a function result to chat history, append a `tool` message with
`tool_name` so the result can be matched to its call:

```json
{"role":"tool","content":"11 degrees celsius","tool_name":"get_weather"}
```

### Size coding integrations

Configure at least 64,000 tokens of context for coding integrations. Local
context allocation consumes memory, and parallel requests multiply that
allocation by the parallel-request count. A derived model can set context
explicitly:

```text
FROM llama3.2
PARAMETER num_ctx 65536
```

```sh
ollama create mymodel
```

### Configure without launching

`ollama launch` normally configures and starts an integration. Add `--config`
to write the integration configuration without starting it:

```sh
ollama launch opencode --config
```

Running bare `ollama` starts the interactive agent and supplies the current
working directory as project context. Use `ollama signin` when its web search
or fetch functionality requires authentication.

### Run local-only

Disable cloud models and web search in `~/.ollama/server.json`:

```json
{
  "disable_ollama_cloud": true
}
```

Alternatively start the server with `OLLAMA_NO_CLOUD=1`. Restart the server and
verify that its logs contain `Ollama cloud disabled: true`.

### Bound concurrency and queueing

Set explicit limits when memory or latency must be predictable:

```sh
OLLAMA_MAX_LOADED_MODELS=2 \
OLLAMA_NUM_PARALLEL=4 \
OLLAMA_MAX_QUEUE=128 \
ollama serve
```

Defaults are three loaded models per GPU, or three for CPU inference; one
parallel request per model; and a queue of 512 before excess work receives
HTTP 503.

### Reduce K/V cache memory

Flash Attention is selected automatically where supported and can be forced
with `OLLAMA_FLASH_ATTENTION=1` or disabled with `0`. When it is enabled,
`OLLAMA_KV_CACHE_TYPE=q8_0` uses roughly half the default `f16` cache memory;
`q4_0` uses roughly one quarter but has greater quality loss.

```sh
OLLAMA_FLASH_ATTENTION=1 OLLAMA_KV_CACHE_TYPE=q8_0 ollama serve
```

## Working method

1. Identify whether the caller uses the native API, a compatibility API, the
   CLI, or an integration; their fields and lifecycle guarantees differ.
2. Inspect `ollama show`, the Modelfile, and server configuration before
   assuming capabilities, context size, or accelerator placement.
3. Budget context and parallelism together because each parallel request
   multiplies context memory.
4. For cloud, search, fetch, or image features, verify authentication, local-only
   policy, and platform support before selecting an implementation.
5. Open the topic reference from the index for full supported values, defaults,
   and endpoint-specific restrictions.
