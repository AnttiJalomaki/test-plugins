# CLI and integrations

## Interactive entry point

Since 0.32, running `ollama` without a subcommand starts an interactive agent
for chat, coding, web features, and delegated work. The current working
directory is passed as project context. Sign in from the CLI when search or
fetch needs authentication. (batch 0.30-0.32)

```sh
ollama
ollama signin
```

## Integration launcher

`ollama launch` configures and starts a selected integration without requiring
manual environment-variable or configuration-file setup. Run it without an
integration name to see the broader selection; the default launcher menu shows
only popular integrations.

```sh
ollama launch
```

Add `--config` to configure an integration without starting it:

```sh
ollama launch opencode --config
```

The former Codex App integration is named ChatGPT and uses the `chatgpt`
launcher. `--restore` returns to the usual ChatGPT profile. (batch 0.30-0.32)

```sh
ollama launch chatgpt
ollama launch chatgpt --restore
```

Imported GGUF models retain tool calling when the artifact supports it. Confirm
the `tools` capability before passing such a model to a supported integration:

```sh
ollama show my-model
ollama launch claude --model my-model
ollama launch hermes --model my-model
ollama launch openclaw --model my-model
```

## Deprecated launcher tags

Launching CodeLlama, Qwen2.5 or Qwen2.5-coder, Llama 3.x, Mistral, StarCoder,
or base DeepSeek-R1 tags produces a deprecation warning before continuing.
(batch 0.30-0.32)

## Context for coding tools

Set the Ollama context length to at least 64,000 tokens for coding integrations.
Recommended local tags are `glm-4.7-flash`, `qwen3-coder`, and `gpt-oss:20b`.
Full-context cloud choices include `glm-4.7:cloud`, `minimax-m2.1:cloud`,
`gpt-oss:120b-cloud`, and `qwen3-coder:480b-cloud`.

At 64K context, `glm-4.7-flash` needs about 23 GB of VRAM locally.

```sh
ollama pull glm-4.7-flash
ollama pull glm-4.7:cloud
```

## Gemma 4

Gemma 4 is available from the model library with the `gemma4` tag:

```sh
ollama run gemma4
```
