# Training and distributed execution

Use this reference for `Trainer`, data collators, tensor/expert/sequence parallelism, distributed correctness, export, checkpoint tying, and optimizer behavior.

## Trainer behavior

### Loss normalization

`TrainingArguments.average_tokens_across_devices` is enabled by default (since 4.54.0). `Trainer` also correctly scales loss when the last gradient-accumulation window contains fewer steps than configured (since 4.55.0). Existing jobs whose number of batches is not divisible by the accumulation factor can have different effective loss scaling after upgrade.

Validate both:

- token-count averaging across ranks;
- normalization of a partial final accumulation window.

### Reproducibility and configuration synchronization

- `DataCollatorForWholeWordMask` accepts a seed, making random whole-word masking reproducible (since 4.51.0).
- At training time, `Trainer` aligns special-token settings in model configuration with the tokenizer (since 4.56.0).
- Youtu-LLM exposes `use_deterministic` for consistent execution (since 5.1.0).

### Evaluation, callbacks, and DDP

- `Trainer` can evaluate under sequence parallelism (since 5.2.0).
- `TrackioCallback` no longer supplies GPU tracking or environment-variable configuration (since 5.2.0).
- `Trainer` exposes `ddp_static_graph` (since 5.6.0).
- Whisper transcription exposes a progress callback for inference monitoring (since 4.55.0).
- `Sam3VideoModel` can disable its progress bar (since 5.1.0).

## Parallel training and model loading

### Tensor parallelism

Accelerate workflows support tensor-parallel training (since 4.50.0).

Decoder-only dense and mixture-of-experts models corrected their all-reduce behavior. Update existing tensor-parallel configuration and checkpoint-conversion mappings rather than assuming earlier plans remain valid (since 5.3.0).

Model-specific capabilities include:

- GPT-OSS 120B supplies a default tensor-parallel plan selectable with `tp_plan="auto"` (since 4.55.0).
- Adapters can be loaded with tensor parallelism (since 5.6.0).
- Gemma 4 MoE has a tensor-parallel plan (since 5.6.0).
- Cohere2Moe's tensor-parallel plan is corrected (since 5.9.0).
- Continuous batching supports tensor parallelism (since 5.9.0).

Quantized tensor-parallel inference has format restrictions; consult the loading reference before composing a quantizer with a plan.

### Expert parallelism

Llama 4 sparse mixture-of-experts layers train with expert parallelism by setting `enable_expert_parallel=True` on `distributed_config`. Experts execute independently across devices and need less communication than tensor parallelism (since 4.54.0).

`gemma-4-26B-A4B-it` supports expert parallelism (since 5.6.0). That release also corrects expert-parallel cases that could silently produce wrong results or NaN loss. Do not trust distributed parity from affected earlier runs without revalidation.

### Sequence parallelism

Llama 4's sequence-parallel path was removed (since 4.56.0). Sequence-parallel evaluation remains available through `Trainer` where supported (since 5.2.0); do not infer model training support from the evaluation feature.

## Distributed correctness and hardware paths

- FSDP cases that produced NaN weights on non-rank-0 processes are corrected (since 5.6.0).
- Static FP8 experts support multi-GPU execution (since 5.4.0).
- XPU provides MegaBlocks MoE kernels, and DiffLlama compile mode works on XPU (since 5.1.0).
- Neuron devices are included in automatic compilation hardware detection (since 5.6.0).
- MUSA exposes TF32 flag support (since 4.57.0).

When validating a distributed job, compare rank-local weights, loss, gradients, and checkpoint round trips—not only rank 0 logs.

## Export and compilation

Torch-exportable decoders accept `inputs_embeds` (since 4.56.0). The Transformers compilation default is `fullgraph=False`, avoiding a strict full-graph requirement that was especially problematic for mixture-of-experts models (since 4.56.0).

TorchScript and `torch.fx` integrations are removed in favor of PyTorch Dynamo and Export (since 5.0.0). Migrate export tooling instead of retaining obsolete configuration flags.

## Weight tying and checkpoint conversion

- Weight tying recurses through every submodel (since 4.56.0).
- Tied weights are tied even when a checkpoint contains both keys with equal values. `.bin` checkpoints containing duplicate tied keys can therefore load differently and require verification (since 5.4.0).
- `WeightConverter` expresses reversible reshape, concatenate, split, quantization, and parallelism mappings (since 5.0.0), and conversion recurses through nested models (since 5.4.0).
- Timm backbones preserve `out_features` across save and load (since 5.2.0).

## Model-specific training behavior

- D-FINE computes auxiliary losses when denoising is disabled, correcting that training objective (since 5.7.0).
- `NemotronHPreTrainedModel` advertises gradient-checkpointing support (since 5.7.0).
- Gemma 4 training accepts text-only samples (since 5.6.0).
- Qwen3.5 includes sequence-classification support (since 5.3.0).
- The OLMo family supplies sequence-classification heads (since 5.6.0).
- Tensor-parallel and checkpoint-conversion mappings for decoder-only dense and MoE models must reflect corrected all-reduce semantics (since 5.3.0).

## Optimizers and removed integrations

`StableAdamW` is available as a training optimizer (since 4.54.0).

The Apex integration, including Apex RMSNorm usage in T5 and related architectures, is removed. Migrate mixed precision and fused operations to native PyTorch equivalents (since 5.8.0).

## Training regression checklist

1. Fix random seeds, data order, and whole-word masking seed.
2. Test final partial accumulation windows and token averaging across ranks.
3. Update tensor-parallel mappings and check every rank's weights and loss.
4. Verify expert-parallel and FSDP runs are free of silent divergence and NaNs.
5. Round-trip tied weights, converted checkpoints, quantized state, and backbone configuration.
6. Re-test export with `inputs_embeds` and the current Dynamo/Export stack.
