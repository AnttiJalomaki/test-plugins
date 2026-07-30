# Releases and runtime operations

## Release-family behavior

The versioned batch `0.30-0.32` introduced the CLI, import, acceleration, and
launcher behavior summarized here and in the model reference.

### Vulkan defaults

From 0.30, Vulkan acceleration is enabled by default. This extends
out-of-the-box GPU acceleration to additional AMD and Intel hardware without
requiring vendor-specific libraries.

### Bare CLI agent

From 0.32, running `ollama` without a subcommand starts an interactive agent
for chat, coding, web features, and delegated work. The agent receives the
current working directory as project context.

```sh
ollama
```

Sign in when web search or fetch requires authentication:

```sh
ollama signin
```

### Launcher rename and deprecations

The former Codex App integration is named ChatGPT:

```sh
ollama launch chatgpt
ollama launch chatgpt --restore
```

`--restore` returns to the usual ChatGPT profile. The default launcher menu
shows only popular integrations; run `ollama launch` to see the broader
selection.

The launcher warns before continuing with CodeLlama, Qwen2.5,
Qwen2.5-coder, Llama 3.x, Mistral, StarCoder, or base DeepSeek-R1 tags because
those integration choices are deprecated.

### MLX load timeout

From 0.32.1, loading an MLX text model honors `OLLAMA_LOAD_TIMEOUT`.

### Withdrawn release

Ollama 0.32.2 was withdrawn. Install or upgrade to 0.32.3 or newer.

### Accelerator and Laguna support

Ollama 0.32.3 adds CUDA support on Windows ARM64 and B200 support through
CUDA 12.

Laguna 2.1 models support chat, thinking, and tool calling. Apple GPU support
for them through the MLX engine arrived in 0.32.4.

## Disable cloud features

For local-only operation, set `disable_ollama_cloud` in
`~/.ollama/server.json`:

```json
{
  "disable_ollama_cloud": true
}
```

Alternatively, start the server with:

```sh
OLLAMA_NO_CLOUD=1 ollama serve
```

Restart after changing the JSON setting. Server logs confirm the effective
state with:

```text
Ollama cloud disabled: true
```

This setting disables both cloud models and web search.

## Concurrent work and queueing

Three environment variables bound server concurrency:

| Setting | Default | Effect |
| --- | --- | --- |
| `OLLAMA_MAX_LOADED_MODELS` | Three times the GPU count, or three for CPU inference | Number of simultaneously loaded models |
| `OLLAMA_NUM_PARALLEL` | One per model | Parallel requests processed by each model |
| `OLLAMA_MAX_QUEUE` | 512 | Queue capacity before excess requests receive HTTP 503 |

```sh
OLLAMA_MAX_LOADED_MODELS=2 OLLAMA_NUM_PARALLEL=4 \
  OLLAMA_MAX_QUEUE=128 ollama serve
```

Parallelism multiplies the model context allocation and its memory requirement
by the number of parallel requests. Include that multiplier when sizing GPU or
system memory.

## Flash Attention and K/V cache memory

Flash Attention is chosen automatically when the hardware supports it. Force
it on or off with:

```sh
OLLAMA_FLASH_ATTENTION=1 ollama serve
OLLAMA_FLASH_ATTENTION=0 ollama serve
```

When Flash Attention is enabled, `OLLAMA_KV_CACHE_TYPE` sets the global cache
default:

| Type | Approximate memory relative to `f16` | Tradeoff |
| --- | --- | --- |
| `f16` | Full | Default quality |
| `q8_0` | One half | Some quality loss |
| `q4_0` | One quarter | Greater quality loss |

```sh
OLLAMA_FLASH_ATTENTION=1 OLLAMA_KV_CACHE_TYPE=q8_0 ollama serve
```

## Exact memory scheduling

New-engine models measure their exact memory requirement before loading rather
than relying on an estimate. This behavior is enabled by default and can:

- avoid memory over-allocation;
- place more of a model on the GPU;
- improve scheduling across multiple or mismatched GPUs; and
- make `ollama ps` memory usage agree more closely with tools such as
  `nvidia-smi`.

At rollout, exact scheduling applied to `gpt-oss`, `llama4`,
`llama3.2-vision`, `gemma3`, `embeddinggemma`, `gemma3n`, `qwen3`,
`qwen2.5vl`, `mistral-small3.2`, `all-minilm`, and other new-engine embedding
models. Other models gain it as they migrate to the new engine.
