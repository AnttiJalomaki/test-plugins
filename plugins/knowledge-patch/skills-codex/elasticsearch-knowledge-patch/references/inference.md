# Inference API

Use this reference for endpoint definitions, request compatibility, chunking,
provider integrations, and operational behavior.

## Endpoint and request controls

### Unified APIs and service paths

In 9.0.0, the Inference API adds unified chat completions, more embedding and
reranking backends, node-local rate limiting, and mTLS for the hosted inference
service. Service paths gain a version prefix and the sparse-embedding route
changes.

### Request fields and chunking

In 9.1.0, EIS sparse-inference request bodies rename `model_id` to `model`.
The Perform Inference API exposes root-level `input_type` for
`text_embedding` and adds common rerank options. Endpoints support `none` to
disable automatic chunking and add a recursive chunker.

In 9.2.0, inference requests gain a configurable query timeout, and chunking
settings no longer have an upper limit. Partial search results are disabled.

### Endpoint deletion and validation

In 9.2.0, invalid inference endpoints can be force-deleted when their model is
invalid or stopping a deployment fails.

### Scaling and aliases

In 9.1.0, adaptive-allocation scale-to-zero is configurable and defaults to 24
hours. Inference services can expose aliases.

### Batching, caching, and late chunking

In 9.3.0, EIS dense and sparse services accept `max_batch_size`, unified
responses report cached tokens, and Jina AI embedding settings support late
chunking.

## Tasks, licenses, and service compatibility

### Expanded task coverage

In 9.3.0, the Inference API adds an embedding task type and EIS completion
support. EIS requires a basic license.

In 9.4.0, Jina AI and Elastic Inference Service add embedding tasks.

### Cohere and general endpoint compatibility

In 9.1.0, new Cohere endpoints use the Cohere V2 API. Cohere gains binary
embeddings, Jina AI accepts an embedding type, and Bedrock Cohere accepts task
settings.

## Provider integrations

### Services added in 9.1.0

The Inference API adds a custom inference service; Vertex AI chat and
completion; Mistral and Hugging Face chat completion; DeepSeek; VoyageAI
embedding and rerank; and SageMaker OpenAI chat and embedding integrations.

### Services and options added in 9.2.0

The Inference API adds ContextualAI reranking; AI21, Google Model Garden
Anthropic, Llama, and IBM watsonx completion and chat support; and Azure AI
reranking. Provider-specific options include Vertex AI embedding dimensions,
custom headers for OpenAI embedding and chat requests, and a Gemini thinking
budget.

### Services added in 9.3.0

Provider coverage expands to Azure OpenAI and Groq chat completion, NVIDIA,
OpenShift AI, and Google Model Garden integrations for Meta, Mistral, Hugging
Face, and AI21 models.

### Services, authentication, and inputs added in 9.4.0

The Inference API adds Fireworks AI chat completion and embeddings, Amazon
Bedrock chat completion, Azure OpenAI custom headers and OAuth2, multimodal
inputs, and reasoning for chat-completion integrations.

Reasoning chat requests no longer accept `max_tokens`. Base64 embedding inputs
must use data-URI format. SageMaker `ElasticTextEmbeddingPayload` requires
`similarity`. Inference timeouts return HTTP 504.
