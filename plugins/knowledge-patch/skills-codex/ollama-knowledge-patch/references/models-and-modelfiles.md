# Models, imports, and Modelfiles

## Import GGUF artifacts

Ollama 0.30 accepts either a GGUF file or a directory containing GGUF files in
a Modelfile's `FROM`.

```text
FROM ./my-model.Q4_K_M.gguf
```

```sh
ollama create -f Modelfile my-model
ollama run my-model
```

Tool calling remains available when the imported GGUF supports it. Verify
`ollama show` lists the `tools` capability before passing it to an integration:

```sh
ollama show my-model
ollama launch claude --model my-model
ollama launch hermes --model my-model
ollama launch openclaw --model my-model
```

## Import Safetensors weights

A Modelfile can build from a directory of Safetensors weights when the
architecture is supported. When the Modelfile is stored with the weights:

```text
FROM .
```

```sh
ollama create my-model
```

Direct import supports Llama, Mistral/Mixtral, Gemma, and Phi-3, including
fine-tunes that have been fused with their foundation model.

## Apply adapters

`ADAPTER` accepts:

- a directory containing a Safetensors adapter; or
- a GGUF adapter file.

The path may be absolute or relative to the Modelfile.

```text
FROM llama3.2
ADAPTER ./adapter.gguf
```

`FROM` must name the exact base used during fine-tuning. A different base can
produce erratic results. For Safetensors adapters, prefer non-quantized
adapters over QLoRA adapters because framework quantization methods differ.

## Quantize during CLI creation

`ollama create` accepts `-q` or `--quantize` for an FP16 or FP32 source model.

```text
FROM /path/to/fp16-model
```

```sh
ollama create --quantize q4_K_M my-model
```

## Declare a minimum version

Use `REQUIRES` to declare the minimum Ollama version needed by a model.

```text
FROM llama3.2
REQUIRES 0.14.0
```

## Use and alias library models

Gemma 4 is available under the `gemma4` tag:

```sh
ollama run gemma4
```

## Run MLX-specific models

On Apple Silicon, the MLX engine supports NVIDIA's model-optimized NVFP4
format, including imported NVFP4 artifacts and dedicated model tags.

The initial Qwen coding preview needs more than 32 GB of unified memory.
Newer MLX tags can be run directly or passed to an integration:

```sh
ollama run qwen3.5:35b-a3b-coding-nvfp4
ollama run gemma4:12b-mlx
ollama launch pi --model gemma4:12b-mlx
```

For digest-addressed uploads and REST quantization, see
[native-api.md](native-api.md). For aliases and compatibility-client context
configuration, see [compatibility-apis.md](compatibility-apis.md).
