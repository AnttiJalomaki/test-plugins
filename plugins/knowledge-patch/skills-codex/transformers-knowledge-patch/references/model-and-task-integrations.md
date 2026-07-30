# Model and task integrations

Use this reference to identify native architectures, supported modalities, task heads, input limits, and integration-specific caveats. Loading, processor, quantization, and distributed constraints remain in their topic references.

## Language and reasoning models

### Dense, hybrid, and encoder-decoder families

- Ernie 4.5 includes a dense 0.3B text model (since 4.54.0).
- LFM2 provides 350M, 700M, and 1.2B models aimed at CPU, GPU, and NPU edge deployment (since 4.54.0).
- DeepSeek V2, Doge, xLSTM, and ModernBERT Decoder are supported. ModernBERT Decoder is causal and handles sequences through 8,192 tokens (since 4.54.0).
- EXAONE 4.0 supports reasoning and non-reasoning modes, tools, and English, Korean, and Spanish (since 4.54.0).
- Arcee, Falcon H1, dots.llm1, SmolLM3, and the encoder-decoder T5Gemma family are supported (since 4.53.0).
- MiniMax-Text-01 supports inference contexts up to four million tokens (since 4.53.0).
- The GLM-4.1V architecture shipped before any corresponding checkpoint was available (since 4.53.0).
- Qwen3 and Qwen3MoE architectures shipped before their weights; DeepSeek-V3 remained work in progress in that release (since 4.51.0).

### Long-context, byte-level, and hybrid-attention models

- Qwen3-Next combines Gated DeltaNet and attention and includes an 80B-A3B checkpoint (since 4.57.0).
- LongCat-Flash provides a 128K context with reasoning and tool use (since 4.57.0).
- BLT uses vocabulary-free UTF-8 byte tokenization with dynamic patching (since 4.57.0).
- OLMo Hybrid interleaves full attention and Gated DeltaNet linear attention; its cache carries KV and recurrent state (since 5.3.0).
- EuroBERT is a bidirectional Llama-like multilingual encoder with an 8,192-token sequence length (since 5.3.0).
- Mistral 4 is a unified instruction/reasoning model with text and image inputs and a 256K context (since 5.4.0).
- HRM-Text is a base autoregressive model without instruction tuning or chat templates. It uses slow abstract-planning and fast detailed-computation recurrent stacks, PrefixLM attention, per-head sigmoid output gates, and parameterless RMSNorm (since 5.9.0).

### Expert and domain-composition models

- GPT-OSS 20B and 120B are mixture-of-experts reasoning models with MXFP4 MoE weights. The 20B form fits in 16 GB; the 120B form fits in 80 GB and has a default tensor-parallel plan (since 4.55.0).
- FlexOlmo exposes independently trained domain experts that can be included or excluded at inference (since 4.57.0).
- OLMO3 and Ministral are supported language-model architectures (since 4.57.0).
- K-EXAONE is supported through EXAONE-MoE (since 5.1.0).
- GLM-5 uses the `GlmMoeDsa` architecture and DeepSeek Sparse Attention (since 5.2.0).
- Qwen3.5 and Qwen3.5 MoE are supported; the first Qwen3.5 checkpoint is a native vision-language model with 397B total and 17B active parameters, hybrid Gated Delta Networks, and sparse MoE layers (since 5.2.0).
- GraniteMoeHybrid is supported as a hybrid mixture-of-experts architecture (since 4.52.1).
- Gemma 4 has pretrained and instruction-tuned 1B, 13B, and 27B variants (since 5.5.0).
- Nemotron 3 is supported, and Qwen3.5 has a sequence-classification head (since 5.3.0).
- Cohere2Moe supports Command A+, with hybrid sliding-window/full attention, shared and routed experts, and a very large context window (since 5.9.0).

### Specialized language architectures

- VaultGemma is a 1B differentially private research checkpoint with a 1,024-token sequence length (since 4.57.0).
- Laguna XS.2 is a MoE decoder whose layers may use different query-head counts while preserving one KV-cache shape. Its sigmoid router combines element-wise gate scores with learned expert bias for auxiliary-loss-free balancing (since 5.7.0).
- SonicMoe is supported as an architecture (since 5.7.0).
- DeepSeek-V4-Flash, DeepSeek-V4-Pro, and Base variants replace Multi-head Latent Attention with hybrid local/long-range attention, use Manifold-Constrained Hyper-Connections instead of residual connections, and apply a static token-to-expert hash table in the first MoE layers (since 5.8.0).
- EXAONE 4.5 is a 33B open-weight vision-language model with a 1.2B visual encoder, 153,600-token vocabulary, contexts through 256K, and Multi-Token Prediction (since 5.8.0).
- Youtu-LLM is a 1.96B model with a 128K context and an optional deterministic-execution mode (since 5.1.0).
- Nemotron-H supports MLP mixers, and Qwen Thinker base checkpoints load without a generative head (since 5.6.0).

## General multimodal and vision-language models

### Image, text, video, and unified generation

- Gemma 3, Aya Vision, Mistral 3.1, SmolVLM2, and SigLIP2 are integrated (since 4.50.0). Aya Vision handles image and text across 23 languages; Mistral 3.1 adds vision and a 128K context; SmolVLM2 supports multiple images and video; SigLIP2 NaFlex preserves variable aspect ratios and resolutions.
- Llama 4 Maverick and Scout load through `Llama4ForConditionalGeneration` and accept text and images. The documented setup installs `transformers[hf_xet]` (since 4.51.0):

```bash
pip install -U 'transformers[hf_xet]'
```

- Phi-4 Multimodal accepts text, images, and audio, produces text, and has a 128K-token context (since 4.51.0).
- Qwen2.5-Omni accepts text, images, audio, and video and streams both text and speech responses (since 4.52.1).
- Janus and Janus-Pro accept images and text and can generate text or images. Select one output mode; interleaved image-and-text output is unsupported (since 4.52.1).
- InternVL3 and MLCD are supported (since 4.52.1).
- Gemma 3n accepts text, image, video, and audio and produces text (since 4.53.0).
- GLM-4.5V, Ovis2, HunYuan, and Seed-OSS are supported (since 4.56.0).
- Qwen3-VL includes dense and MoE Instruct and Thinking variants; Qwen3 Omni MoE provides unified multimodal generation (since 4.57.0).
- LFM2-VL accepts text and variable-resolution images. It preserves native resolution through 512×512 and patches larger images; the 1.6B variant also uses a thumbnail (since 4.57.0).
- Ernie 4.5 VL MoE is supported (since 5.2.0); its later model/config names follow vLLM and SGLang conventions (since 5.3.0).

### Vision-language understanding and action

- Command A Vision supports captioning, visual question answering, and document/chart understanding through the Cohere2 Vision integration (since 4.55.0).
- DeepSeek VL consumes images and text and returns text (since 4.54.0).
- PerceptionLM performs image and video understanding (since 4.54.0).
- PI0 adds vision-language-action inference, generating robot actions from visual observations and language instructions (since 5.4.0).
- Music Flamingo handles audio-language understanding and reasoning across speech, sound, and music for audio sequences up to 20 minutes (since 5.5.0).
- AudioFlamingoNext checkpoints are supported (since 5.9.0).

## Vision, segmentation, detection, and matching

### Segmentation and grounding

- SAM-HQ provides promptable high-quality segmentation, but its integration does not support fine-tuning (since 4.52.1).
- EoMT is pipeline-compatible for image segmentation (since 4.54.0) and can use a DINOv3 backbone (since 5.1.0).
- Florence-2 supports prompted captioning, detection, and segmentation; SAM 2 supports point- or box-prompted image and video segmentation (since 4.56.0).
- EdgeTAM provides mobile-oriented real-time video segmentation (since 4.57.0).
- VidEoMT performs online video segmentation (since 5.4.0).
- SAM3-LiteText combines SAM3's ViT-H image encoder with a compact distilled MobileCLIP text encoder (since 5.6.0).
- MM Grounding DINO performs zero-shot object grounding and detection with original MM Grounding DINO or LLMDet checkpoints (since 4.55.0).

### Detection and visual features

- D-FINE object detection and BitNet are supported (since 4.52.1).
- DINOv3 provides dense visual features; MetaCLIP 2 provides multilingual image-text representations (since 4.56.0).
- DEIMv2 offers real-time object detection in eight sizes from X through Atto. Larger versions use a Spatial Tuning Adapter to turn single-scale DINOv3 features into multiple scales; the smallest use pruned HGNetv2 backbones (since 5.7.0).
- AIMv2 vision encoders and EoMT are supported (since 4.54.0).

### Feature matching and geometry

- LightGlue matches local features between image pairs and can pair with SuperPoint for pose or homography estimation (since 4.53.0).
- EfficientLoFTR produces image correspondences for pose or homography estimation (since 4.54.0).
- Native LightGlue loading does not use remote code; remove `trust_remote_code=True` and load it normally (since 5.5.0).

## Document, OCR, and retrieval models

### Layout analysis and recognition

- PP-DocLayoutV3 predicts instance-segmented document layout and reading order. Pair it with GLM-OCR for a two-stage layout-analysis then parallel-recognition workflow (since 5.1.0).
- PP-DocLayoutV2 combines RT-DETR element detection/classification with pointer-network reading-order prediction (since 5.3.0).
- GLM-OCR recognizes complex documents (since 5.1.0).
- Qianfan-OCR is a 4B end-to-end image-to-text document model for structured parsing, tables, charts, document QA, and key-information extraction. Layout-as-Thought emits structured layout before the final answer (since 5.6.0).
- Kosmos-2.5 performs spatially grounded OCR and image-to-Markdown conversion (since 4.56.0).

### Document geometry, tables, and formulas

- UVDoc rectifies single or batched document images (since 5.4.0).
- SLANeXt recognizes table structure; PP-OCRv5 mobile/server detector and recognizer families cover multilingual document and scene text (since 5.4.0).
- PPLCNet supplies document-orientation, table, and text-line orientation classifiers, while PPLCNetV3 is a CPU-oriented vision backbone (since 5.4.0).
- SLANet and SLANet_plus provide lightweight CPU-oriented table-structure recognition for documents and natural scenes (since 5.6.0).
- PP-FormulaNet-L and PP-FormulaNet_plus-L perform lightweight image-to-text recognition of formulas and table structures in document and natural-scene images (since 5.8.0).

### Visual-document representation and retrieval

- ColQwen2 encodes document pages as images and returns multi-vector embeddings for late-interaction retrieval (since 4.53.0).
- ModernVBert combines ModernBERT with a SigLIP vision encoder for visual-document understanding and retrieval (since 5.3.0).
- ColModernVBert produces ColPali-style multi-vector document-image embeddings for relevance scoring (since 5.3.0).
- Granite 4.1 Vision targets chart, table, and semantic key-value extraction. It combines a SigLIP2 encoder with Window Q-Former projectors and injects visual features at eight language-model points (since 5.8.0).

## Speech, audio, and acoustic-token models

### Speech recognition

- Kyutai STT uses streaming-codec recognition. `kyutai/stt-1b-en_fr` is bilingual; `kyutai/stt-2.6b-en` is English-only (since 4.53.0).
- Parakeet provides CTC recognition through `ParakeetForCTC` (since 4.57.0). Parakeet TDT is a distinct additional integration (since 5.9.0).
- Moonshine supports streaming (since 5.1.0).
- VoxtralRealtime performs low-latency incremental ASR over arriving audio chunks (since 5.2.0).
- VibeVoice ASR processes up to 60 minutes of 24 kHz audio with hotwords, joint transcription, diarization, timestamps, 50+ languages, and code-switching (since 5.3.0).
- Granite Speech Plus provides prompted speech-to-text with speaker annotations and word timestamps. Its projector concatenates final speech-encoder states with a configurable subset of intermediate states (since 5.8.0).

### Audio-language and speech generation

- CSM provides contextual text-to-speech (since 4.52.1).
- Dia provides dialogue-oriented speech synthesis with nonverbal cues and audio conditioning (since 4.53.0).
- Voxtral extends Ministral models with audio for transcription, translation, audio Q&A/summarization, and voice-driven function calls. `mistralai/Voxtral-Mini-3B-2507` and `mistralai/Voxtral-Small-24B-2507` have 32K contexts and handle up to 30 minutes of transcription or 40 minutes of broader audio understanding (since 4.54.0).
- Higgs Audio V2 supports one or multiple speakers and zero-shot voice cloning. Its separate 24 kHz tokenizer encodes speech, music, and sound at 25 fps without diffusion (since 5.3.0).
- Music Flamingo supports audio-language reasoning over clips through 20 minutes (since 5.5.0).

### Acoustic tokenization

- X-Codec provides semantic-aware audio tokenization, music continuation, and text-to-sound synthesis (since 4.56.0).
- VibeVoice Acoustic Tokenizer supports continuous audio-token designs for long-form multi-speaker synthesis (since 5.2.0).

## Embeddings, moderation, time series, and science

### Embeddings and classification

- ModernBERT has a question-answering module (since 4.51.0).
- DeepSeek V3 provides `DeepseekV3ForTokenClassification` (since 4.57.0).
- NomicBERT is a native dense-embedding model with an 8,192-token context for search, clustering, and classification; inputs use task-specific instruction prefixes (since 5.5.0).
- Jina Embeddings v3 provides multilingual task-specific embeddings with five built-in LoRA adapters and sequences through 8,192 tokens (since 5.4.0).
- OpenAI Privacy Filter performs bidirectional token classification for on-premises PII detection and masking in one forward pass. It emits per-token probabilities over eight privacy categories and coherent decoded spans (since 5.6.0).
- Llama Guard 4 and ShieldGemma 2 provide multimodal moderation and image-safety category classification, respectively (since 4.52.1 and 4.50.0).

### Forecasting and depth

- TimesFM supports time-series forecasting, and `AutoModelForTimeSeriesPrediction` is directly importable (since 4.52.1 and 4.53.0):

```python
from transformers import AutoModelForTimeSeriesPrediction
```

- TimesFM 2.5 is a decoder-only zero-shot forecaster with continuous quantile prediction (since 5.3.0).
- Prompt Depth Anything produces metric depth maps using an iPhone LiDAR prompt (since 4.50.0).

### Scientific and earth-observation tasks

- EVOLLA generates protein-language output from protein sequences, structures, and user queries (since 4.54.0).
- CHMv2 estimates forest-canopy height from high-resolution optical satellite imagery (since 5.4.0).
- V-JEPA 2 supplies video-encoder and action-conditioned world-model support (since 4.53.0).
- Distill Any Depth is integrated (since 4.51.0).

## Integration selection checklist

1. Confirm the task head exists, not only the base architecture.
2. Check modality, context/audio/image limits, and processor normalization.
3. Note architecture-only or work-in-progress integrations without released weights.
4. Verify unsupported training or output modes such as SAM-HQ fine-tuning and Janus interleaving.
5. Consult the loading and multimodal references for native-vs-remote loading, cache, device, and preprocessing requirements.
