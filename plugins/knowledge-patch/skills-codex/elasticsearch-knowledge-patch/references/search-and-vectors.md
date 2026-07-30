# Search, semantic fields, and vectors

Use this reference to choose field mappings, retrievers, quantization, and
rescore behavior. Validate defaults on newly created indices separately from
existing mappings.

## Semantic fields and sparse retrieval

### `semantic_text` availability and query support

The `semantic_text` field type is generally available in 8.18.0. It integrates
field mapping and inference for semantic search.

In 9.0.0, `semantic_text` works with `match`, `sparse_vector`, and kNN vector
queries and with highlighting. It supports multi-fields and participates in
text-family behavior.

### Mapping and chunking controls

In 9.1.0, `semantic_text` mappings accept `index_options`, configurable
chunking, and bit vectors. Compatible models default new fields to BBQ. Empty
content skips embedding generation; `semantic_text` subfields are excluded
from field-capabilities responses; and `sparse_vector` mappings gain a default
token-pruning setting.

An existing `semantic_text` mapping can update `inference_id` in 9.3.0. New
fields default to ELSER on the Elastic Inference Service when available, and
semantic text supports BFloat16.

In 9.4.0, new `semantic_text` fields inherit DiskBBQ indexing and BFloat16
storage. The default inference ID and model switch to Jina v5. The
text-similarity rank retriever selects chunking defaults suited to its
inference ID.

### Sparse-vector token pruning

Token pruning for `sparse_vector` queries is generally available in 8.19.0
rather than a technical-preview feature.

### Chunk and embedding retrieval

In 9.2.0, the Fields API can retrieve indexed `semantic_text` chunks,
semantic embeddings can be included in `_source`, and semantic queries can
span multiple inference IDs.

## Multi-vector and dense-vector mappings

### `rank_vectors`

In 8.18.0, the experimental `rank_vectors` field type supports multi-vector
late-interaction ranking for dense-vector second-stage reranking, including
ColBERT- and ColPali-style models. It suits workloads where HNSW indexing is
too costly but reranking improves relevance.

In 9.3.0, `rank_vectors` supports BFloat16.

### Quantized vector mapping controls

In 9.1.0, `rescore_vector` is generally available. `oversample: 0` bypasses
oversampling and rescoring, while BBQ indices receive a default oversample
value. Quantized index types gain `vector_rescore`; existing `dense_vector`
mappings can be updated to `bbq_flat` or `bbq_hnsw`.

### DiskBBQ

In 9.2.0, the `disk_bbq` `dense_vector` index type targets lower-memory
operation without HNSW's memory profile. It accepts only floating-point
vectors, uses one-bit quantization, and is not recommended for low-dimensional
vectors. Tune kNN with `num_candidates` or `visit_percentage`.

```http
PUT vectors
{"mappings":{"properties":{"vector":{"type":"dense_vector","index_options":{"type":"disk_bbq"}}}}}
```

In 9.4.0, new indices default vector indexing to DiskBBQ. Its quantization
level is configurable at 1, 2, 4, or 7 bits. HNSW dense-vector fields add
`flat_index_threshold`.

### BFloat16 and on-disk rescoring

In 9.3.0, every `dense_vector` index type can store
`element_type: bfloat16`, halving stored bytes per value at the cost of reduced
precision and conversion overhead. `on_disk_rescore: true` keeps raw-vector
rescoring on disk when vectors exceed available RAM.

```http
PUT vectors
{"mappings":{"properties":{"vector":{"type":"dense_vector","element_type":"bfloat16","index_options":{"type":"disk_bbq","on_disk_rescore":true}}}}}
```

### Vector input and indexing

In 9.3.0, vectors can be indexed from base64 and HNSW early termination is
enabled by default. `semantic_text` can optionally use GPU indexing for HNSW
and `int8_hnsw`.

In 9.2.0, the new `GPUPlugin` supports indexing vectors on a GPU.

## Vector source and memory behavior

### Storage outside `_source`

In 9.0.0, `sparse_vector` values can be stored outside `_source`.
Synthetic-source indices also have an index setting to skip recovery source.

### Retrieval and reindexing

In 9.1.0, source retrieval can include or exclude vectors. Node and index stats
expose dense-vector off-heap usage. Painless `dotProduct` and
`cosineSimilarity` can use float vectors with byte-vector fields.

In 9.2.0, reindex always includes vectors despite transparent removal of
vectors from ordinary `_source` results.

## Retrievers and rank fusion

### Rescorer and linear retrievers

In 9.0.0, search adds a generic rescorer retriever based on request rescoring
and a linear retriever for weighted sums of sub-retrievers. Quantized kNN
vectors can be rescored, and BBQ indices are generally available.

In 9.1.0, the linear retriever adds `l2_norm` normalization and a minimum
score. Search also gains a pinned retriever and simplified Linear and RRF
retrievers. Text-similarity reranking can optionally be allowed to fail.

### RRF weighting and multi-index search

In 9.2.0, RRF retrievers support weighting, and simplified RRF syntax supports
per-field weights. Simplified RRF and linear retrievers can query multiple
indices; the linear retriever adds a top-level normalizer.

### MMR diversification

In 9.3.0, search adds a retriever for result diversification using MMR. In
9.4.0, the MMR retriever accepts `semantic_text`.

## Rescoring and query execution

### Scripted and contextual rescoring

In 9.2.0, search adds a script-based rescorer. The
`text_similarity_reranker` can use `chunk_rescorer` to split fields and score
contextual snippets instead of reranking the entire field.

### Direct I/O for BBQ rescoring

In 9.2.0, setting the Java option `vector.rescoring.directio=true` on every
vector-search node makes BBQ rescoring use direct I/O. This avoids severe
page-cache latency when off-heap memory is scarce, but slows searches when
vectors fit in memory.

### Filters, profiling, and query-vector construction

In 9.2.0, kNN filters support nested metadata, and profiling supports HNSW kNN
queries with early termination.

In 9.4.0, search adds an embedding query-vector builder.
