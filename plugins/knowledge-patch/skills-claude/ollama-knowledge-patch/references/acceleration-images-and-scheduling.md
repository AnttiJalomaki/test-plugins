# Acceleration, images, and scheduling

## Vulkan default

Since 0.30, Vulkan acceleration is enabled by default, extending
out-of-the-box GPU acceleration to more AMD and Intel hardware without
vendor-specific libraries. (batch 0.30-0.32)

## Accelerator and model-engine additions

Since 0.32.3, CUDA support includes Windows ARM64 and B200 through CUDA 12.
Laguna 2.1 models support chat, thinking, and tool calling. Since 0.32.4,
Laguna also has Apple GPU support through the MLX engine.
(batch 0.30-0.32)

Since 0.32.1, MLX text-model loading honors `OLLAMA_LOAD_TIMEOUT`.
(batch 0.30-0.32)

## MLX models on Apple Silicon

The MLX engine supports NVIDIA's model-optimized NVFP4 format on Apple Silicon,
including imported NVFP4 models and dedicated tags. The initial Qwen coding
preview requires more than 32 GB of unified memory.

Run newer MLX tags directly or pass them to an integration:

```sh
ollama run qwen3.5:35b-a3b-coding-nvfp4
ollama run gemma4:12b-mlx
ollama launch pi --model gemma4:12b-mlx
```

## Image generation from the CLI

On macOS, pass a prompt directly to an image model. Ollama saves the result in
the current directory; terminals that support image rendering also show an
inline preview.

```sh
ollama run x/z-image-turbo "a watercolor lighthouse in a winter storm"
ollama run x/flux2-klein "a neon sign reading OPEN 24 HOURS"
```

Windows and Linux were not supported when this experimental workflow was
announced.

## Interactive image controls

Within an image-model session, set dimensions with `/set width` and
`/set height`:

```text
/set width 1024
/set height 768
```

Each model provides a recommended default step count. Image generation also
supports reproducible random seeds and negative prompts.

## Exact memory scheduling

New-engine models measure their exact memory requirement before loading rather
than relying on an estimate. The behavior is enabled by default and:

- avoids memory over-allocation;
- can place more of a model on the GPU;
- improves scheduling across multiple or mismatched GPUs; and
- makes `ollama ps` memory usage agree with utilities such as `nvidia-smi`.

At rollout, exact measurement applied to `gpt-oss`, `llama4`,
`llama3.2-vision`, `gemma3`, `embeddinggemma`, `gemma3n`, `qwen3`,
`qwen2.5vl`, `mistral-small3.2`, `all-minilm`, and other new-engine embedding
models. Additional model families gain it as they migrate to the new engine.
