# Tokenizers, processors, and multimodal inputs

## Migrate tokenizer construction

Transformers 5.0.0 unifies tokenizer implementations behind
`TokenizersBackend`, `SentencePieceBackend`, `PythonBackend`, or
`MistralCommonBackend`. `AutoTokenizer.from_pretrained()` still chooses the
backend automatically. `PythonBackend` replaces the former
`PreTrainedTokenizer` role for custom Python tokenizers, and
`PreTrainedTokenizerBase` is the minimal backend-independent API.

- A tokenizer backed by `tokenizers` can be constructed blank for training or
  directly from `vocab` and `merges` (5.0.0).
- Constructors do not accept a `vocab_file` path. Use `from_pretrained()` for
  file-based loading (5.0.0).
- `AutoTokenizer` no longer guesses tokenizer type from a model directory name
  when config lacks `model_type` as of 5.2.0. Repositories that relied on a
  substring such as `bert` must declare the model type.
- A later class-selection change that chose the wrong tokenizer for models such
  as DeepSeek R1 was reverted in 5.7.0.
- `PreTrainedTokenizerFast` skips `clean_up_tokenization` for BPE tokenizers as
  of 5.7.0.
- DeepSeek OCR automatic tokenizer mapping selects the intended class in 5.8.0.

```python
from transformers import LlamaTokenizer

blank = LlamaTokenizer()
tokenizer = LlamaTokenizer(vocab=vocab, merges=merges)
```

## Migrate tokenizer calls and subclass contracts

- `encode_plus()` is deprecated in 5.0.0; call the tokenizer.
- `decode()` handles single and batched inputs in 5.0.0, so a batched value does
  not require `batch_decode()`.
- `apply_chat_template()` returns `BatchEncoding` in 5.0.0. Select fields such
  as `input_ids`; do not assume a raw tensor or list of token IDs.
- `sanitize_special_tokens()` and base-class target-mode helpers including
  `as_target_tokenizer()` were removed in 5.0.0. Use `text_target=`.
- `prepare_seq2seq_batch()` is deprecated and `BatchEncoding.words()` is
  replaced by `word_ids()` in 5.0.0.
- Custom tokenizer subclasses no longer inherit working base implementations of
  `create_token_type_ids_from_sequences`, `prepare_for_model`,
  `build_inputs_with_special_tokens`, or `truncate_sequences` in 5.0.0.
  Implement them or use `PythonBackend`.
- `Siglip2Tokenizer` enforces the preprocessing defaults used during training
  as of 5.1.0.

```python
encoded = tokenizer(["hello", "world"])
texts = tokenizer.decode(encoded["input_ids"])
chat = tokenizer.apply_chat_template(messages, return_tensors="pt")
input_ids = chat["input_ids"]

model_inputs = tokenizer(
    source_texts,
    text_target=target_texts,
    max_length=128,
    return_tensors="pt",
)
model_inputs["labels"] = model_inputs.pop("input_ids_target")
```

## Save special tokens and templates

- New tokenizer saves consolidate named special tokens into
  `tokenizer_config.json` and added tokens into `tokenizer.json` beginning in
  5.0.0. Older `special_tokens_map.json` and `added_tokens.json` remain readable
  but are no longer written.
- `special_tokens_map` contains only named attributes. Put additional values in
  `extra_special_tokens`; `additional_special_tokens` is converted for
  compatibility, and extended special-token accessors were removed (5.0.0).
- Multiple raw chat-template files can be saved and loaded as of 4.52.1.
- `apply_chat_template()` can prefill custom fields such as
  `reasoning_content` and `thinking` as of 5.9.0.
- `Trainer` synchronizes special-token settings from the tokenizer into model
  configuration at training time as of 4.56.0.

## Build multimodal chat content

- Chat templates can load audio from video inputs in 4.51.0.
- `apply_chat_template()` accepts in-memory video values, not only paths and
  URLs, in 4.55.0.
- `ProcessorMixin.apply_chat_template()` correctly loads PIL image inputs in
  4.57.0.
- OpenAI-style `image_url` content entries are accepted by
  `apply_chat_template()` as of 5.2.0.
- `make_batched_video()` accepts five-dimensional arrays in 5.1.0.
- Multimodal callers can pass `mm_token_type` as non-padded lists in 5.4.0.
- The standard embedding input name is plural `inputs_embeds` in 5.2.0; rename
  integrations that still accept `input_embeds`.
- Vision-language models share a Qwen2-VL-derived 3D position-ID interface from
  5.3.0. Update custom processors or code that builds position IDs manually for
  affected Ernie and GLM4V models.

## Select and extend image/video processors

- Most vision and vision-language models gain torch/torchvision-backed fast
  processors for CPU and CUDA in 4.52.1.
- Fast processors expand to SuperPoint, SegFormer, Janus, DeepSeek-VL, and
  DeepSeek-VL Hybrid in 4.55.0.
- Video processors are separate classes beginning in 4.52.1.
- In 5.4.0, separate `BaseImageProcessor` and `BaseImageProcessorFast` classes
  are replaced by one backend architecture. The
  `image_processing_utils_fast` module is removed; custom processors migrate to
  `image_processing_utils`.
- `AutoProcessor.from_pretrained()` forwards Hub keyword arguments as of 5.4.0
  rather than silently discarding them.
- PIL-backed image processors no longer incorrectly require `torchvision` in
  5.5.0, so PIL-only preprocessing works without that dependency.

## Respect model-specific preprocessing

- `Dinov2ForImageClassification` handles checkpoints with register tokens
  correctly as of 4.52.1.
- PerceptionLM receives video input and preprocesses non-tiled images correctly
  in 4.56.0. The same release repairs Fuyu image inference, Qwen-VL video beam
  search, LLaVA-OneVision batch inference, and tensor-device placement for
  Idefics2, Idefics3, and SmolVLM.
- Image-text inference supports batch sizes above one in 4.57.0.
- `WhisperFeatureExtractor` keeps `input_features` and `attention_mask` at
  consistent lengths in 4.57.0.
- Fast `center_crop` matches the slow implementation as of 4.57.0.
- LFM2-VL preserves native image resolution through 512×512 without forced
  upscaling or aspect-ratio distortion. Larger inputs are divided into 512×512
  patches, and the 1.6B model also gets a global thumbnail (4.57.0).
- `BeitImageProcessorFast.reduce_label` returns `labels`, not `label`, and Janus
  resizing rounds dimensions rather than truncating them in 5.1.0.
- `Llava-OneVision` accepts `image_sizes` in 5.1.0.

### Gemma 4 image budget

Gemma 4's processor in 5.5.0 preserves aspect ratio while targeting one of 70,
140, 280, 560, or 1,120 soft tokens per image; 280 is the default. Total pixels
must fit the selected patch budget, and processed height and width must each be
divisible by 48.

Do not apply ordinary ImageNet mean/std normalization. Gemma 4 patch embeddings
perform their final scaling to `[-1, 1]` internally.

### Full versus pooled text embeddings

`text_embeds` for SAM3, EdgeTAM, and SAM3-Lite-Text expects full text embeddings
as of 5.9.0. Callers that pass pooler output must switch to the unpooled
sequence representation.

## Make masking reproducible

`DataCollatorForWholeWordMask` accepts a seed in 4.51.0. Set it explicitly when
tests or training restarts require identical random whole-word masks.
