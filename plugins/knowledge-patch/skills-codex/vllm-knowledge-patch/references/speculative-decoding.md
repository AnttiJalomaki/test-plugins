# Speculative Decoding

## Compatibility progression

- In `0.7-0.10`, structured output and thinking became compatible with
  speculative decoding while the old V0 speculative workers were removed.
- In `0.15-0.18`, speculation added Unified Parallel Drafting,
  structured-output compatibility, `min_tokens`, Mamba cache-align support,
  GPU N-gram execution, and async-scheduling compatibility.
- In `0.19-0.22`, async scheduling and speculation gained zero-bubble overlap.
  A draft model can select an MoE backend through `--speculative-config`
  (`-sc`); CPU draft execution, thinking-budget awareness, an independent
  drafter attention backend, and a custom callable proposer were added.
- In `0.23-0.26`, speculation added causal DFlash, dynamic execution,
  FlashInfer DFlash, universal heterogeneous-vocabulary speculation through
  TLI plus DSpark drafters, runtime draft-weight update, hybrid
  sliding/full-attention DFlash, and a separate `kv_cache_dtype` inside
  `speculative_config`.

## Suffix decoding

Use `method="suffix"` for draft-model-free dynamic-depth speculation. Defaults
are tree depth 24, a 10,000-request global cache, speculation factor 1.0, and
minimum token probability 0.1. A zero cache limit disables shared cache state.

```bash
vllm serve MODEL --speculative-config '{
  "method": "suffix",
  "num_speculative_tokens": 8,
  "suffix_decoding_max_cached_requests": 0
}'
```

## N-gram lookup

If both `prompt_lookup_min` and `prompt_lookup_max` are absent they default to
5. If only one is supplied, the omitted bound copies it, selecting an exact
lookup width.

```bash
vllm serve MODEL --speculative-config '{
  "method": "ngram",
  "num_speculative_tokens": 4,
  "prompt_lookup_min": 3
}'
```

## Custom proposers

For an experimental backend, use `method="custom_class"` and put the fully
qualified proposer class in `model`. It is instantiated with `VllmConfig` and
must implement `propose`.

```python
llm = LLM(
    model="target-model",
    speculative_config={
        "method": "custom_class",
        "model": "my_package.MyProposer",
    },
)
```

## Gemma 4 assistants

Gemma 4 assistant checkpoints use `method="mtp"`, not the generic draft-model
path. If startup resolves one to `draft_model`, upgrade to a release with
Gemma 4 MTP support.

```python
speculative_config={
    "method": "mtp",
    "model": "gemma-4-assistant-checkpoint",
}
```

## Heterogeneous vocabularies

`use_heterogeneous_vocab=True` is valid only with `method="draft_model"`.
Initialization intersects normalized token strings and proposals are limited
to shared tokens. Probabilistic draft sampling is unsupported.

```python
speculative_config={
    "method": "draft_model",
    "model": "different-tokenizer-draft-model",
    "num_speculative_tokens": 3,
    "use_heterogeneous_vocab": True,
}
```

## Draft execution and acceptance

- Use `draft_tensor_parallel_size`, not `tensor_parallel_size`, inside
  `speculative_config`.
- `max_model_len` in that object applies to the draft; `temperature` and
  `top_p` remain sampling parameters.
- `parallel_drafting` is limited to EAGLE and draft-model methods.
- Rejection sampling accepts `strict` (default), `probabilistic`, or
  `synthetic`. `synthetic_acceptance_rate` must be within `[0, 1]`.

```python
speculative_config={
    "method": "draft_model",
    "model": "draft-model",
    "draft_tensor_parallel_size": 2,
    "parallel_drafting": True,
    "rejection_sample_method": "synthetic",
    "synthetic_acceptance_rate": 0.8,
}
```

## Output guarantees

Rejection sampling preserves the target distribution up to numerical
precision, and greedy output is validated against non-speculative decoding.
It does not guarantee identical token log probabilities: precision, hardware,
and batch composition can still change probabilities or outputs.
