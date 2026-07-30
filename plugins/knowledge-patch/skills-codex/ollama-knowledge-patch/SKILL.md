---
name: ollama-knowledge-patch
description: Ollama
version: 0.32.5
license: MIT
metadata:
  author: Nevaberry
---


# Ollama

Use this skill when installing or operating Ollama, creating or importing
models, integrating coding tools, or implementing native and compatible API
clients. Check the project's installed Ollama version and its actual model
metadata before applying version-sensitive advice.
## Reference index

| Reference | Topics |
| --- | --- |
| [releases-and-runtime.md](references/releases-and-runtime.md) | Upgrade hazards, CLI behavior, accelerators, server limits, cache memory, scheduling |
| [models-and-modelfiles.md](references/models-and-modelfiles.md) | GGUF and Safetensors imports, adapters, quantization, requirements, context, MLX |
| [native-api.md](references/native-api.md) | Thinking, tool history, blobs, model creation, embeddings |
| [compatibility-apis.md](references/compatibility-apis.md) | Anthropic Messages, OpenAI chat/completions/embeddings/Responses, aliases |
| [cloud-integrations-and-web.md](references/cloud-integrations-and-web.md) | Launchers, coding context, cloud tags, web search/fetch, MCP |
| [image-generation.md](references/image-generation.md) | Native and compatible image APIs, CLI generation, interactive settings |

## Critical compatibility notes

- Do not deploy the withdrawn `0.32.2` release. Upgrade to `0.32.3` or newer.
- The former Codex App launcher entry is now ChatGPT. Use
  `ollama launch chatgpt`; add `--restore` to return to the usual ChatGPT
  profile.
- Launching CodeLlama, Qwen2.5 or Qwen2.5-coder, Llama 3.x, Mistral,
  StarCoder, or base DeepSeek-R1 tags emits a deprecation warning.
- `/api/embed` replaces `/api/embeddings`. It accepts one string or a list and
  returns an embeddings matrix.
- `/v1/responses` is stateless: do not send `previous_response_id`,
  `conversation`, or `truncation`. Carry conversation state in the
  application.
- The OpenAI-compatible endpoints implement documented subsets, not every
  upstream request option. Validate endpoint-specific unsupported fields.
- Image generation and `/v1/images/generations` are experimental. The
  compatibility endpoint requires `response_format: "b64_json"`.

## Choose the right interface

Use the native API for Ollama-specific controls:

- `POST /api/chat` for multi-turn messages, tools, and separated thinking.
- `POST /api/generate` for text or automatically detected image models.
- `POST /api/embed` for one or many embedding inputs.
- `POST /api/create` for model creation, uploaded files, adapters, and
  quantization.

Use a compatibility API when an existing client already speaks that protocol:

- Point Anthropic Messages clients at `http://localhost:11434`.
- Point OpenAI clients at `http://localhost:11434/v1/`.
- Supply the API key or auth token required by the client; the local server
  ignores its value.

Use the CLI when configuring integrations, importing local artifacts, or
starting an interactive agent.

## Upgrade and launch safely

Running `ollama` without a subcommand starts the interactive agent. It receives
the current directory as project context and can handle chat, coding, web
features, and delegated work.

```sh
ollama
ollama signin
```

`ollama launch` configures and starts an integration. Add `--config` when only
configuration should be written.

```sh
ollama launch
ollama launch opencode --config
ollama launch chatgpt --restore
```

The default launcher menu shows popular integrations; the bare
`ollama launch` command exposes the broader selection.

## Import and create models

A Modelfile `FROM` may name a GGUF file, a directory containing GGUF files, or
a supported Safetensors directory.

```text
FROM ./my-model.Q4_K_M.gguf
```

```sh
ollama create -f Modelfile my-model
ollama run my-model
```

For Safetensors weights stored next to the Modelfile:

```text
FROM .
```

```sh
ollama create my-model
```

Use `ADAPTER` for a Safetensors adapter directory or GGUF adapter file. The
`FROM` model must be the exact fine-tuning base.

```text
FROM llama3.2
ADAPTER ./adapter.gguf
```

Quantize an FP16 or FP32 source during creation with `-q` or `--quantize`.

```sh
ollama create --quantize q4_K_M my-model
```

Use `REQUIRES` in a Modelfile when the model needs a minimum Ollama version.
See [models-and-modelfiles.md](references/models-and-modelfiles.md) for
supported direct-import architectures, API uploads, and quantization choices.

## Preserve tool and thinking semantics

Before using an imported GGUF with an integration, confirm its metadata exposes
the `tools` capability.

```sh
ollama show my-model
ollama launch claude --model my-model
```

Native `/api/chat` and `/api/generate` accept `think` as a boolean or an effort
value: `"low"`, `"medium"`, `"high"`, or `"max"`. Chat output puts reasoning
in `message.thinking`.

```json
{
  "model": "gpt-oss:20b",
  "messages": [{"role": "user", "content": "Solve this carefully."}],
  "think": "high",
  "stream": false
}
```

When replaying an executed function result, append a `tool` message with
`tool_name`.

```json
{"role":"tool","content":"11 degrees celsius","tool_name":"get_weather"}
```

Tool calls themselves may arrive in streamed responses.

## Size context for coding tools

Coding integrations should receive at least 64,000 tokens of context. Local
recommendations include `glm-4.7-flash`, `qwen3-coder`, and `gpt-oss:20b`;
full-context cloud choices include `glm-4.7:cloud`,
`minimax-m2.1:cloud`, `gpt-oss:120b-cloud`, and
`qwen3-coder:480b-cloud`.

At 64K context, `glm-4.7-flash` needs about 23 GB of local VRAM. If an
OpenAI-compatible client cannot set context size, create a derived model:

```text
FROM llama3.2
PARAMETER num_ctx 65536
```

```sh
ollama create mymodel
```

## Control server memory and queueing

The server defaults are:

- `OLLAMA_MAX_LOADED_MODELS`: three times the GPU count, or three on CPU.
- `OLLAMA_NUM_PARALLEL`: one request per model.
- `OLLAMA_MAX_QUEUE`: 512 queued requests, then HTTP 503.

Parallel requests multiply the model's allocated context and memory. Bound all
three settings deliberately for constrained hosts.

```sh
OLLAMA_MAX_LOADED_MODELS=2 OLLAMA_NUM_PARALLEL=4 \
  OLLAMA_MAX_QUEUE=128 ollama serve
```

Flash Attention is selected automatically when supported. Force it with
`OLLAMA_FLASH_ATTENTION=1` or disable it with `0`. With it enabled,
`OLLAMA_KV_CACHE_TYPE=q8_0` uses roughly half the default `f16` cache memory;
`q4_0` uses roughly one quarter, with greater quality loss.

```sh
OLLAMA_FLASH_ATTENTION=1 OLLAMA_KV_CACHE_TYPE=q8_0 ollama serve
```

## Keep local-only deployments local

Disable cloud models and web search with either server configuration or an
environment variable, then restart:

```json
{
  "disable_ollama_cloud": true
}
```

```sh
OLLAMA_NO_CLOUD=1 ollama serve
```

Confirm the logs contain `Ollama cloud disabled: true`.

## Cloud and hosted web operations

Sign in before using cloud tags. They work with normal `run`, `pull`, `ls`, and
`cp` commands, while inference executes on ollama.com. After a pull, local APIs
and client libraries address the cloud tag like any other model.

```sh
ollama signin
ollama pull gpt-oss:120b-cloud
```

Hosted search and fetch use bearer authentication at
`https://ollama.com/api/web_search` and
`https://ollama.com/api/web_fetch`. The Python and JavaScript clients expose
matching helpers that can be passed directly as chat tools. Give standalone
search agents about 32K context or more because results can be large.

## Verify before handoff

- Run `ollama show <model>` and verify required capabilities.
- Confirm the effective context size and multiply it by configured parallelism
  when estimating memory.
- Reject unsupported compatibility fields before relying on their behavior.
- Treat experimental image endpoints as changeable and retain decoded image
  handling for base64 results.
- Use `ollama ps` alongside accelerator tools to verify actual placement and
  memory after loading.
