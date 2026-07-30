# Serving APIs, Operations, and Security

## OpenAI-compatible surface

### Early endpoint and request expansion (`0.7-0.10`)

The server added Jina- and Cohere-compatible reranking in 0.7, `/score` for
embedding models and streaming transcription in 0.8, and a Responses API in
0.10. The request `model` field became optional in 0.8.

Request controls added during this period include
`return_tokens_as_token_id`, image embeddings, V1 `allowed_token_ids`, Chat
`mm_processor_kwargs`, `bad_words`, and chunking for audio longer than 30
seconds. `LLM.chat` accepts `chat_template_kwargs`; embedding requests gained
truncation, usage can report `cached_tokens`, and 0.10 adds
`tokenization_kwargs` for embedding truncation.

V1 gained structured and reasoning output in 0.8, including backend-specific
guided-decoding options. `--enable-reasoning` was deprecated in 0.9.
Structured output later became compatible with thinking and speculative
decoding; XGrammar gained `tool_choice: required`, Guidance gained structural
tags, and 0.10 tool calling supports required choice and `$defs`.

### Responses and tools (`0.11-0.14`)

The Responses API added MCP tools, multi-turn non-Harmony requests,
reasoning-item input, parsed tool arguments, `parallel_tool_calls` compliance,
tool filtering, Browser/Container MCP tools, extra body parameters, and MCP
streaming.

Version 0.14 adds `reasoning_effort` and
`--default-chat-template-kwargs`. Parser coverage expanded through SeedOSS,
Qwen3-Coder XML, DeepSeek-V3.2, Gigachat 3, Holo2, FunctionGemma, and GLM-4.7.

### Responses and Anthropic compatibility (`0.15-0.18`)

Responses adds partial-message generation, `include_stop_str_in_output`, and
`prompt_cache_key` in 0.15; sampling parameters and returned token IDs in 0.16;
structured output and streamed reasoning-part events in 0.17; and streamed
tool/function calls in 0.18.

Anthropic-compatible serving adds thinking blocks, token counting, and
`tool_choice=none` in 0.17, followed by redacted thinking blocks in 0.18.

### New controls (`0.19-0.22`)

Version 0.19 adds `/v1/chat/completions/batch` and permits multiple embedding
types in one call. Version 0.21 supports streamed required or named tool
choices, `system_fingerprint`, and `prompt_embeds` content parts. Version 0.22
adds `chat_template_kwargs` to Responses, configurable truncation side for
OpenAI-compatible endpoints, and `thinking_token_budget` to Completions;
`reasoning_effort` maps to `enable_thinking`.

Explicit `/start_weight_update` and `/finish_weight_update` endpoints arrived
in 0.21 for reinforcement-learning integrations.

### Parser and endpoint growth (`0.23-0.26`)

Version 0.23 unifies reasoning and tool-call parsing under `Parser.parse`.
Versions 0.24-0.25 add the Streaming Parser Engine and parsers for Qwen3,
MiniMax-M2, Nemotron V3, Kimi K2.5-K2.7, Seed-OSS, and DeepSeek V4. Strict tool
calling works in both Chat Completions and Responses.

Version 0.24 adds strict tool calling for supported Qwen3, MiniMax-M2, and
Nemotron V3 parsers and lets `/v1/embeddings` accept messages plus
`chat_template_kwargs`. Version 0.25 adds Responses namespace tools,
per-request timing metrics, and `return_loss_mask`. Version 0.26 adds
`bad_words` to `/v1/completions` and exposes `logprob_token_ids` on the
Python-backed endpoints.

Anthropic-compatible messages add structured output, effort, and system-role
messages inside the messages array in 0.23. Version 0.24 reports cache usage and
handles system messages in the middle of a conversation.

Version 0.26 introduces an endpoint-plugin framework and adds
`/abort_requests` to the reinforcement-learning development API router.

## Explicit compatibility limits (`engine-and-openai-server`)

The Completions `suffix` field and Chat `image_url.detail` are unsupported.
Chat's `user` field is accepted but ignored. `parallel_tool_calls=false`
guarantees at most one tool call. The default `true` allows multiple calls but
does not guarantee them because the behavior remains model-dependent.

The Responses service supports creation plus retrieval and cancellation:

```text
POST /v1/responses
GET  /v1/responses/{response_id}
POST /v1/responses/{response_id}/cancel
```

## Embedding, scoring, and speech

### Embeddings and transcription (`0.11-0.14`)

Version 0.12 adds Whisper `verbose_json` and timestamps. Version 0.13 adds
embedding responses with `encoding_format=bytes_only` and multiple images or
audio items per request. Version 0.14 adds `continue_final_message` to
`/embeddings`.

### Speech and realtime (`0.15-0.18`)

Version 0.15 exposes `avg_logprob` and `compression_ratio` in Whisper
`verbose_json` segments. Version 0.16 adds batch transcription and translation,
0.17 adds automatic language detection, and 0.18 adds beam search to offline
and online transcription.

Version 0.16 introduces WebSocket streaming audio through the Voxtral realtime
infrastructure. Version 0.17 extends realtime streaming to Qwen3-ASR.

### Scoring and pooling IO (`0.15-0.18`)

Version 0.15 adds BGE-M3 sparse and ColBERT embeddings. `/score` accepts either
`data_1`/`data_2` or `queries`/`documents`. Later releases add sparse-embedding
IO processing, multimodal late-interaction scoring, and Cohere Embed v2
compatibility.

## Rendering and preprocessing

Version 0.15 adds prompt-preprocessing render endpoints. Version 0.18 adds
`vllm launch render`, allowing preprocessing/rendering to run as a GPU-less
service apart from GPU inference.

## Alternative frontends and transports

### gRPC

Version 0.14 adds a gRPC server as an alternative to REST, using a binary
protocol and HTTP/2 multiplexing. Despite later TLS and mTLS support in the Rust
frontend, the 0.25 guidance calls the gRPC interface insecure and suitable only
for private use.

### Rust frontend (`0.19-0.22`, `0.23-0.26`)

An experimental in-tree Rust frontend and data-parallel integration arrived in
0.22; API-key authorization also extended to `/v2` endpoints.

Version 0.23 adds streaming `generate`, dynamic LoRA endpoints,
`/server_info`, and `--enable-request-id-headers`. Version 0.24 adds API-key
auth, CORS, `/pause`, `/resume`, `/is_paused`, and `/get_world_size`. Version
0.25 adds static HTTPS and mTLS for HTTP and gRPC. Version 0.26 adds video,
audio, and a native `vllm-bench`; TorchCodec becomes a video-decoding backend
in 0.25.

## Server operations and configuration

### Health and introspection (`0.7-0.10`, `0.11-0.14`)

Version 0.8 adds `/load` statistics, `/is_sleeping`, and live HTTP-server SSL
key rotation. Version 0.10 adds `get_tokenizer_info` for tokenizer and
chat-template details.

From 0.11, health returns HTTP 503 when the engine is dead. Version 0.13 adds
`/reset_prefix_cache` for KV connectors, and 0.14 adds `/server_info` for
environment information.

### Server controls (`0.15-0.18`)

Version 0.15 can derive `api_server_count` from data-parallel size and adds
`--ssl-ciphers`. Version 0.16 supports nested YAML configuration,
`--disable-access-log-for-endpoints`, and clearing multimodal and encoder
caches. Version 0.18 adds `--distributed-timeout-seconds` plus a graceful
shutdown timeout for in-flight requests.

### Environment and timeout controls (`0.19-0.22`)

`VLLM_MAX_N_SEQUENCES` in 0.19 enforces a sequence-count limit.
`VLLM_MEDIA_CACHE` in 0.20 opts into media-URL caching.
`VLLM_SKIP_MODEL_NAME_VALIDATION` in 0.21 bypasses model-name validation.
Version 0.22 adds `--cpu-distributed-timeout-seconds`.

## Logging and metrics

### Metrics migration (`0.7-0.10`)

The metrics `vllm:time_in_queue_requests`,
`vllm:model_forward_time_milliseconds`, and
`vllm:model_execute_time_milliseconds` were deprecated in 0.8.
`--show-hidden-metrics-for-version` exposes hidden metrics during migration.
Later releases added `vllm:cache_config_info`, KV-event publishing, and access
to in-memory Prometheus metrics. Version 0.10 deprecated `gpu_` prefixes on
metrics that are not GPU-specific.

Version 0.15 removes `vllm:time_per_output_token_seconds`; use
`vllm:inter_token_latency_seconds`.

### Request and engine logs (`engine-and-openai-server`)

`--enable-log-requests` logs request IDs, parameters, and LoRA requests at INFO;
at DEBUG it also logs prompt text or token IDs. `VLLM_LOGGING_LEVEL` selects the
threshold. `--aggregate-engine-logging` reports aggregate rather than
per-engine data-parallel statistics. `--fail-on-environ-validation` converts
environment-validation failures into errors.

## CLI behavior

Version 0.8 introduced the consolidated `vllm bench` command and deprecated
`--dataset` in `benchmark_serving.py`. Version 0.10 adds paged help with
`--help=page` and changes the CLI default model to Qwen3-0.6B.

## Request validation and security

### Cache isolation and serialization (`0.7-0.10`)

Version 0.9 adds prefix-cache salting, `VLLM_ALLOW_INSECURE_SERIALIZATION`, and
a V1 option that disables pickle fallback. Version 0.10 permits Completions and
Responses requests to carry `cache_salt`. Reproducible prefix-cache hashing can
use SHA-256 plus CBOR.

### Security corrections (`0.11-0.14`, `0.15-0.18`)

Version 0.11 addresses GHSA-wr9h-g72x-mwhm, 0.13 adds protection for
CVE-2025-62164, and 0.14 prevents tokens from leaking through crash logs and
loads PyTorch weights with `weights_only=True`. Version 0.14 also corrects
invalid UTF-8 token handling, CPU RoPE output under `--enforce-eager`, tool-call
stream completion, CPU scheduling stalls after encoder-cache leaks,
tools-plus-`response_format` crashes, and Voxtral transcription.

Version 0.18 makes NemotronVL and KimiK25 honor `trust_remote_code` and gates
reinforcement-learning weight-sync deserialization behind the insecure
serialization setting.

### Strict validation (`0.23-0.26`)

Serving rejects out-of-vocabulary token IDs; non-positive parallel and
scheduling values; non-finite temperature or repetition penalties; degenerate
structured-output configuration; and per-request selection of a GPU video
backend. Invalid image URLs return HTTP 422. Regex and derender work are
resource-bounded.
