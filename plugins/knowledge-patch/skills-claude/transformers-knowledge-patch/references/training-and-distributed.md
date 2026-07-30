# Training and distributed execution

## Choose a parallel strategy

- Accelerate workflows support tensor-parallel training from 4.50.0.
- Llama 4 sparse MoE layers support expert-parallel training in 4.54.0. Set
  `enable_expert_parallel=True` on `distributed_config`; experts execute on
  separate devices and require less communication than tensor parallelism.
- Llama 4's sequence-parallel path was removed in 4.56.0.
- `Trainer` can evaluate under sequence parallelism as of 5.2.0.
- Corrected all-reduce handling for dense and MoE decoder-only models in 5.3.0
  requires old tensor-parallel configs and checkpoint conversion mappings to be
  updated together.
- Static FP8 experts work in multi-GPU configurations as of 5.4.0.
- Adapters can load with tensor parallelism in 5.6.0. Gemma 4 MoE has a
  tensor-parallel plan, and `gemma-4-26B-A4B-it` supports expert parallelism.
- Continuous batching gains tensor parallelism in 5.9.0; this is independent of
  ordinary `generate()` tensor-parallel execution.

## Audit Trainer behavior

- `TrainingArguments.average_tokens_across_devices` defaults to enabled in
  4.54.0. Include the resulting cross-device token denominator in loss
  comparisons.
- `Trainer` corrected loss scaling for a final gradient-accumulation window
  containing fewer steps than configured in 4.55.0. Training behavior changes
  when batch count is not divisible by accumulation steps.
- `Trainer` aligns model-config special tokens with tokenizer settings during
  training as of 4.56.0.
- `Trainer` supports evaluation while sequence parallelism is active in 5.2.0.
- `Trainer` exposes `ddp_static_graph` in 5.6.0.
- 5.6.0 fixes expert-parallel cases that could silently return wrong results or
  NaN loss, plus FSDP cases that wrote NaN weights on non-rank-0 processes.
  Re-run correctness checks when upgrading affected training jobs.
- D-FINE computes auxiliary losses when denoising is disabled as of 5.7.0,
  correcting the objective for that configuration.
- `NemotronHPreTrainedModel` advertises gradient-checkpointing support in 5.7.0.

## Optimizers and model training modes

- `StableAdamW` is available as a training optimizer beginning in 4.54.0.
- SAM-HQ fine-tuning is not supported in 4.52.1 even though inference and
  promptable segmentation are integrated.
- Gemma 4 accepts text-only training samples as of 5.6.0.
- Youtu-LLM exposes `use_deterministic` for consistent execution in 5.1.0.
- `Sam3VideoModel` can disable its progress bar in 5.1.0.
- VITS forward calls accept `speaking_rate` in 5.3.0.

```python
outputs = model(**inputs, speaking_rate=1.2)
```

## Compilation, export, and reusable components

- Transformers compilation defaults to `fullgraph=False` in 4.56.0, avoiding a
  restrictive full-graph requirement, especially for MoE models.
- Torch-exportable decoders accept `inputs_embeds` as of 4.56.0.
- Weight tying recurses through every nested submodel in 4.56.0.
- DiffLlama can use compile mode on XPU in 5.1.0, and XPU gains a MegaBlocks MoE
  kernel implementation in the same release.
- Attention and Experts components are reusable standalone modules as of 5.1.0.
- Neuron devices join the automatic compilation hardware list in 5.6.0.

## Preserve backbone and integration state

- Timm backbones retain `out_features` across save/load as of 5.2.0.
- `TrackioCallback` no longer provides GPU tracking or environment-variable
  configuration in 5.2.0; remove reliance on either behavior.
- Backbones in 5.1.0 are created from `config.backbone_config`, and pretrained
  weights load only when present in the checkpoint.
- T5Gemma2 propagates the selected attention implementation into every
  subconfiguration, including `config.encoder.text_config`, in 5.1.0.
- EoMT can use a DINOv3 backbone beginning in 5.1.0.

## Distributed upgrade checks

1. Verify exact loss denominators on one full and one partial accumulation
   window.
2. Compare single-device output with tensor/expert-parallel output on fixed
   inputs, including an MoE checkpoint.
3. Assert every rank's weights remain finite after an FSDP save boundary.
4. Rebuild conversion mappings whenever tensor-parallel collective placement
   changes.
5. Confirm compilation is actually active on the intended hardware; support in
   the automatic hardware list does not guarantee every model graph compiles.
