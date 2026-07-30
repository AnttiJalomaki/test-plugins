# Speculative Decoding

## Compatibility progression

### `0.7-0.10`

V1's structured output became compatible with speculative decoding. Llama 4
with EAGLE joined model coverage. The old V0 speculative workers were removed
in 0.10.

### `0.11-0.14`

Version 0.14 rejects unsupported speculative sampling parameters rather than
silently ignoring them. Async scheduling is enabled by default in 0.14 except
for speculation other than MTP/Eagle, as well as pipeline parallelism and CPU.
Model Runner V2 gained `min_tokens` and other sampling controls but remained
experimental.

### `0.15-0.18`

Version 0.16 adds Unified Parallel Drafting and structured-output
compatibility. Version 0.17 adds `min_tokens` and speculation for Mamba cache
alignment. Version 0.18 moves NGram speculation to the GPU and permits it with
async scheduling. Model Runner V2 adds probabilistic rejection sampling during
this range.

### `0.19-0.22`

Version 0.19 combines async scheduling and speculation with zero-bubble overlap
and permits a draft-specific MoE backend in `--speculative-config`/`-sc`.
Version 0.20 adds CPU draft-model speculation. Version 0.21 honors thinking
budgets and permits the drafter to use an independent attention backend.
Version 0.22 accepts a custom callable proposer.

Model Runner V2 adds multimodal speculative embeddings, greedy/logprob modes
for rejection sampling, and hybrid support during this range. DeepSeek V4 gains
MTP speculation.

### `0.23-0.26`

Version 0.23 adds causal DFlash. Version 0.24 adds dynamic speculation and a
FlashInfer-backed DFlash path. Version 0.25 adds TLI universal speculation
across heterogeneous vocabularies and DSpark drafters, plus dynamic speculation
under full CUDA graphs.

Version 0.26 adds runtime draft-weight updates, hybrid
sliding-window/full-attention DFlash drafters, and a draft-side
`kv_cache_dtype` inside `speculative_config`.

## CLI and configuration mechanics (`engine-and-openai-server`)

`--spec-method`, `--spec-model`, and `--spec-tokens` fill the corresponding
fields of `--speculative-config`. A shorthand and the JSON object cannot both
set the same field. Automatic speculator detection is skipped for cloud-storage
model URIs; configure those models explicitly.

Draft execution uses `draft_tensor_parallel_size`, not
`tensor_parallel_size`, inside `speculative_config`. Draft `max_model_len`
also applies to the draft model, while `temperature` and `top_p` remain sampling
parameters.

`parallel_drafting` is limited to EAGLE and draft-model methods. Rejection
sampling accepts `strict` (default), `probabilistic`, or `synthetic`.
`synthetic_acceptance_rate` must be in `[0, 1]`.

```python
speculative_config = {
    "method": "draft_model",
    "model": "draft-model",
    "draft_tensor_parallel_size": 2,
    "parallel_drafting": True,
    "rejection_sample_method": "synthetic",
    "synthetic_acceptance_rate": 0.8,
}
```

## Suffix decoding (`speculative-decoding`)

`method="suffix"` performs dynamic-depth speculation without a draft model.
Defaults are tree depth 24, a 10,000-request global cache, speculation factor
1.0, and minimum token probability 0.1. A cache limit of zero disables the
global cache.

```bash
vllm serve MODEL --speculative-config '{
  "method": "suffix",
  "num_speculative_tokens": 8,
  "suffix_decoding_max_cached_requests": 0
}'
```

## N-gram lookup (`speculative-decoding`)

If both `prompt_lookup_min` and `prompt_lookup_max` are absent, each defaults to
5. If only one is given, the omitted bound copies it. Providing one bound
therefore selects an exact lookup width.

```bash
vllm serve MODEL --speculative-config '{
  "method": "ngram",
  "num_speculative_tokens": 4,
  "prompt_lookup_min": 3
}'
```

NGram execution moved to the GPU in 0.18 and became compatible with async
scheduling.

## Custom proposer classes (`speculative-decoding`)

For the experimental class-based backend, select `method="custom_class"` and
place the fully qualified class name in `model`. The class is constructed from
a `VllmConfig` and must implement `propose`.

```python
llm = LLM(
    model="target-model",
    speculative_config={
        "method": "custom_class",
        "model": "my_package.MyProposer",
    },
)
```

Version 0.22 also accepts a custom callable proposer.

## MTP and Gemma 4 (`speculative-decoding`)

Configure a Gemma 4 assistant checkpoint as `method="mtp"`, not as a generic
draft model. If startup resolves it to `draft_model`, upgrade to a build with
Gemma 4 MTP support rather than forcing the generic route.

```python
speculative_config = {
    "method": "mtp",
    "model": "gemma-4-assistant-checkpoint",
}
```

## Heterogeneous vocabularies (`speculative-decoding`)

`use_heterogeneous_vocab=True` is valid only with `method="draft_model"`.
Initialization intersects normalized token strings and limits proposals to
shared tokens. Probabilistic draft sampling is unsupported, so this mode is
greedy-only.

```python
speculative_config = {
    "method": "draft_model",
    "model": "different-tokenizer-draft-model",
    "num_speculative_tokens": 3,
    "use_heterogeneous_vocab": True,
}
```

TLI provides a newer universal-speculation path across heterogeneous
vocabularies from 0.25, alongside DSpark drafters.

## Accuracy expectations (`speculative-decoding`)

Rejection sampling preserves the target distribution up to numerical
precision, and greedy decoding is validated against non-speculative decoding.
That losslessness does not make token log probabilities stable. Hardware
precision and batch composition can still change probabilities or outputs
across runs.
