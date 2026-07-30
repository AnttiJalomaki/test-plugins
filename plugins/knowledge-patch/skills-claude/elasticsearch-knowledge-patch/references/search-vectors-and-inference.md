# Search, Vectors, Reranking, and Inference

Use this reference when choosing vector mappings, semantic retrieval,
retrievers, rescorers, or Inference API endpoints. Benchmark with the actual
dimensions, data distribution, memory budget, and provider.

## Contents

- [Semantic field mappings](#semantic-field-mappings)
- [Dense-vector storage and indexing](#dense-vector-storage-and-indexing)
- [Vector retrieval and inspection](#vector-retrieval-and-inspection)
- [Retriever and rescorer selection](#retriever-and-rescorer-selection)
- [Inference request behavior](#inference-request-shape-and-endpoint-behavior)
- [Inference tasks and providers](#inference-tasks-and-providers)

## Semantic field mappings

### `semantic_text`

`semantic_text` is generally available as of 8.18.0 and combines field mapping
with inference for semantic search.

- It participates in the text field family and supports `match`,
  `sparse_vector`, and kNN queries, highlighting, and multi-fields (9.0.0).
- Mappings support `index_options`, configurable chunking, and bit vectors in
  9.1.0. Compatible models default new fields to BBQ.
- Empty content skips embedding generation.
- `semantic_text` subfields are excluded from field-capabilities responses.
- The Fields API can retrieve indexed semantic chunks in 9.2.0, and semantic
  embeddings can be explicitly included in `_source`.
- Semantic queries can span multiple inference IDs (9.2.0).
- Existing mappings can update `inference_id` in 9.3.0.
- New fields default to ELSER on Elastic Inference Service when available in
  9.3.0, and BFloat16 is supported.
- HNSW and `int8_hnsw` semantic fields can optionally use GPU indexing
  (9.3.0).
- In 9.4.0, new fields inherit DiskBBQ indexing and BFloat16 storage defaults,
  and the default inference ID/model changes to Jina v5.
- The text-similarity rank retriever chooses chunking defaults suited to its
  inference ID in 9.4.0.

Review defaults when upgrading rather than assuming an existing mapping and a
newly created mapping behave identically.

### `sparse_vector`

- Sparse-vector values can be stored outside `_source` in 9.0.0.
- Sparse-vector mappings gain a default token-pruning setting in 9.1.0.
- Sparse-query token pruning is generally available as of 8.19.0.
- The Inference API sparse-embedding service route changed in 9.0.0.

### `rank_vectors`

The experimental `rank_vectors` type in 8.18.0 stores multiple vectors for
late-interaction second-stage ranking with models such as ColBERT and ColPali.
It is useful when indexing all token vectors into HNSW is too expensive but
reranking can improve relevance. BFloat16 is supported in 9.3.0.

## Dense-vector storage and indexing

### Quantization and rescoring

- BBQ vector indices are generally available in 9.0.0.
- Quantized kNN results can be rescored in 9.0.0.
- `rescore_vector` is generally available in 9.1.0.
- `oversample: 0` disables oversampling and rescoring; BBQ indices otherwise
  receive a default oversample value.
- Quantized index types add `vector_rescore`, and existing `dense_vector`
  mappings can be updated to `bbq_flat` or `bbq_hnsw` in 9.1.0.
- HNSW early termination is enabled by default in 9.3.0.
- HNSW fields add `flat_index_threshold` in 9.4.0.

### DiskBBQ

The `disk_bbq` dense-vector index type introduced in 9.2.0 targets
lower-memory operation without HNSW's memory profile:

```http
PUT vectors
{"mappings":{"properties":{"vector":{"type":"dense_vector","index_options":{"type":"disk_bbq"}}}}}
```

It initially accepts only floating-point vectors, uses one-bit quantization,
and is not recommended for low-dimensional data. Tune queries with
`num_candidates` or `visit_percentage`.

In 9.4.0, new vector indices default to DiskBBQ and its quantization level can
be configured to 1, 2, 4, or 7 bits.

### BFloat16 and on-disk raw vectors

All dense-vector index types can store `element_type: bfloat16` in 9.3.0,
halving bytes per value at the cost of precision and conversion overhead.
Set `on_disk_rescore: true` when raw vectors exceed available RAM:

```http
PUT vectors
{"mappings":{"properties":{"vector":{"type":"dense_vector","element_type":"bfloat16","index_options":{"type":"disk_bbq","on_disk_rescore":true}}}}}
```

Vector values can also be indexed from base64 as of 9.3.0.

### Direct I/O

On every vector-search node, the 9.2.0 Java option
`vector.rescoring.directio=true` makes BBQ rescoring use direct I/O. It avoids
severe page-cache latency under low off-heap memory but is slower when vectors
fit in memory.

In 9.1.0 specifically, the default `true` can make `bbq_hnsw` search up to ten
times slower for in-memory vectors. Set it to `false` on every search node and
restart; remove that override in 9.1.1.

### GPU indexing

A `GPUPlugin` can index vectors on a GPU as of 9.2.0. In 9.3.0,
`semantic_text` can optionally use GPU indexing for HNSW and `int8_hnsw`.
Mixed-GPU 9.3.1 clusters have a usage-reporting log-flood issue; see the
compatibility reference.

## Vector retrieval and inspection

- New indices exclude vectors from `_source` by default.
- Source retrieval can explicitly include or exclude vectors in 9.1.0.
- Semantic embeddings can be explicitly included in `_source`, and the Fields
  API can retrieve indexed `semantic_text` chunks in 9.2.0.
- Reindex always includes vectors even when transparent source-vector removal
  hides them from `_source`.
- Node and index stats expose dense-vector off-heap memory usage in 9.1.0.
- kNN filters accept nested metadata in 9.2.0.
- HNSW kNN profiling includes early termination in 9.2.0.
- Painless `dotProduct` and `cosineSimilarity` accept float vectors against
  byte-vector fields as of 9.1.0.
- Search adds an embedding query-vector builder in 9.4.0.

## Retriever and rescorer selection

### Fusion and normalization

- A generic rescorer retriever based on request rescoring and a linear
  retriever for weighted sums arrive in 9.0.0.
- The linear retriever gains `l2_norm` normalization and `min_score` in
  9.1.0, alongside simplified Linear and RRF retriever forms.
- RRF retrievers support weights in 9.2.0; simplified RRF supports per-field
  weights.
- Simplified RRF and linear retrievers can query multiple indices in 9.2.0.
- The linear retriever gains a top-level normalizer in 9.2.0.

### Specialized retrievers

- A pinned retriever arrives in 9.1.0.
- Text-similarity reranking can be allowed to fail in 9.1.0.
- Script-based rescoring arrives in 9.2.0.
- `text_similarity_reranker.chunk_rescorer` chunks fields and scores
  contextual snippets instead of an entire field in 9.2.0.
- An MMR retriever for result diversification arrives in 9.3.0.
- In 9.4.0, the MMR retriever accepts `semantic_text`, and ES|QL offers an MMR
  command.

## Inference request shape and endpoint behavior

### Request and chunking options

- The Perform Inference API puts `input_type` at the request root for
  `text_embedding` in 9.1.0 and adds common rerank options.
- EIS sparse-inference request bodies rename `model_id` to `model` in 9.1.0.
- Set chunking to `none` to disable automatic chunking, or select the recursive
  chunker (9.1.0).
- Configured chunking settings no longer have an upper limit in 9.2.0.
- Inference requests gain a configurable query timeout in 9.2.0, while partial
  search results are disabled.
- Invalid endpoints can be force-deleted when the model is invalid or stopping
  a deployment fails (9.2.0).
- EIS dense and sparse services accept `max_batch_size` in 9.3.0.
- Jina AI embedding settings support late chunking in 9.3.0.
- Unified responses report cached tokens in 9.3.0.
- Base64 embedding inputs must use data-URI format in 9.4.0.
- SageMaker `ElasticTextEmbeddingPayload` requires `similarity` in 9.4.0.
- Inference timeouts return HTTP 504 in 9.4.0.

### Scaling, compatibility, and security

- Node-local rate limiting, mTLS for the hosted inference service, unified chat
  completions, version-prefixed service paths, and additional embedding and
  reranking backends arrive in 9.0.0.
- Adaptive-allocation scale-to-zero is configurable in 9.1.0 and defaults to
  24 hours.
- New Cohere endpoints use the Cohere V2 API in 9.1.0.
- Inference services can expose aliases in 9.1.0.
- EIS requires a basic license as of 9.3.0.
- Request-level `secret_parameters` overrides are rejected in 9.3.8 and
  9.4.4.

## Inference tasks and providers

Provider-specific configuration is not portable. Validate task type,
capabilities, request fields, authentication, timeout behavior, and model
compatibility against the selected service.

### Added in 9.1.0

- A custom inference service.
- Vertex AI chat/completion.
- Mistral and Hugging Face chat completion.
- DeepSeek.
- VoyageAI embedding and reranking.
- SageMaker OpenAI-compatible chat and embeddings.
- Cohere binary embeddings.
- Jina AI embedding-type selection.
- Bedrock Cohere task settings.

### Added or expanded in 9.2.0

- ContextualAI and Azure AI reranking.
- AI21, Google Model Garden Anthropic, Llama, and IBM watsonx
  completion/chat.
- Vertex AI embedding dimensions.
- Custom headers for OpenAI embedding and chat requests.
- Gemini thinking-budget configuration.

### Added or expanded in 9.3.0

- A general embedding task type and EIS completion.
- Azure OpenAI and Groq chat completion.
- NVIDIA and OpenShift AI.
- Google Model Garden integrations for Meta, Mistral, Hugging Face, and AI21.

### Added or expanded in 9.4.0

- Fireworks AI chat completion and embeddings.
- Amazon Bedrock chat completion.
- Jina AI and Elastic Inference Service embedding tasks.
- Azure OpenAI custom headers and OAuth2.
- Multimodal inputs and reasoning for chat-completion integrations.
- Reasoning chat requests no longer accept `max_tokens`.
