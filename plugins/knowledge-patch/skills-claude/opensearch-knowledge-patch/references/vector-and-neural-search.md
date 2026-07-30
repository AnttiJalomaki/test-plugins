# Vector and Neural Search

Use this reference for k-NN mappings, Faiss and Lucene execution, vector
compression, semantic fields, sparse retrieval, reranking, and highlighting.

## Engine and mapping compatibility

### Faiss becomes the default k-NN engine

In 2.18, the implicit k-NN engine changes from NMSLIB to Faiss. With
`space_type: "cosinesimil"` and no explicit engine, indexing normalizes vectors
and stored values differ from the inputs. Already-normalized vectors can use
`innerproduct` for equivalent scoring without that implicit normalization.

### Vector engine and nested-search support

Since 2.19.0, Lucene supports binary vector indexes, and Faiss supports cosine
similarity for k-NN and radial search without caller-side normalization. Nested
k-NN `inner_hits` can return multiple values with Lucene or Faiss. NMSLIB adds
`expand_nested_docs`, and neural k-NN queries add `expand_nested`.

### k-NN mapping and request behavior

A training-backed vector mapping cannot specify both a model ID and
`dimension`; when a model ID is present, its training index supplies the
dimension. Since 2.19.0, `index.knn` is immutable after index creation,
searches using `fields` work with `knn_vector`, and `rescore: false` actually
disables rescoring.

### Nested text-embedding field maps

In 2.19, the `text_embedding` processor stops substituting nested
`_ingest._value` paths. Map the nested input path directly:

```json
"field_map": {
  "books.title": "title_embedding"
}
```

### k-NN compatibility validation

Since 3.1.0, derived vector source cannot be enabled with `index.knn: false`.
Mode and compression settings are rejected for indexes created before 2.17.0.

### Semantic-field tuning

Since 3.2.0, semantic fields can configure engine, mode, compression level,
method, and other parameters of their generated dense `knn_vector`. They also
support ingest batch sizing, sparse-encoding prune strategies, configurable
chunking, reuse of existing embeddings, and `TOKEN_ID` sparse embeddings.

## Vector source, retrieval, and storage

### Experimental derived vector source

In 2.19.0, disabled-by-default derived source removes k-NN vectors from stored
JSON `_source` and reinjects them when a document is read. It supports flat
vector mappings, object fields, and single-level nested fields.

### Search scoring and k-NN defaults

Since 3.0.0, the default scorer is `BM25Similarity` rather than
`LegacyBM25Similarity`, and concurrent segment search is enabled by default for
k-NN. Derived vector source is production-ready with Faiss, Lucene, and NMSLIB.

### k-NN vector retrieval through doc values

Since 3.7.0, `docvalue_fields` retrieves float, byte, and binary `knn_vector`
values from Lucene and Faiss indexes at every compression level without
reindexing. The default representation is base64-encoded binary rather than an
array. Remote index builds also support 1-bit scalar quantization.

## Build, compression, and memory

### Vector index build and rescore defaults

GPU-accelerated index builds are production-ready in 3.1.0. Remote builds are
enabled by default with `index.knn.remote_index_build.enabled`. New OnDisk
indexes using 4x compression rescore by default; set `rescore: false` to retain
the earlier behavior.

### Expanded GPU vector indexing

Since 3.2.0, GPU-accelerated index building supports FP16, byte, and binary
vectors in addition to FP32.

### Remote vector-build failure behavior

Since 3.2.0, a terminal remote-vector-index-build failure does not fall back to
a CPU build.

### Compressed-vector recall controls

Since 3.2.0, binary-quantized Faiss indexes can use asymmetric distance
computation, comparing a full-precision query with compressed document vectors,
and random rotation to reduce information loss at 32x compression. Asymmetric
distance computation also works when Lucene graph search operates on Faiss
indexes.

### One-bit vector quantization

Since 3.6.0, Lucene BBQ and Faiss scalar quantization support 1-bit vectors at
32x compression for approximate and exact search. Coverage includes Lucene flat
format and Faiss memory-optimized search, and Faiss 32x compression defaults to
the SQ 1-bit encoder. Vector metadata can use Zstandard compression, and byte
vectors have a Hamming-distance scorer.

### QAT-accelerated compression

OpenSearch Custom Codecs adds Intel QAT-accelerated Zstandard compression in
3.1.0.

### Cross-plugin custom codecs

Since 3.5.0, the AdditionalCodecs registration path lets custom codecs serve
plugins including k-NN, Neural Search, and Security Analytics.

## Faiss and Lucene execution

### Faiss and k-NN administration

Since 3.0.0, Faiss explain supports exact, ANN, radial, and disk-based k-NN
search. `memory-optimized-search` provides a lower-memory Faiss mode, and
node-level circuit breakers support heterogeneous memory limits. Legacy
index-level settings for `ef_construction`, `m`, space type, and plugin
enablement are removed.

### Faiss execution options

Since 3.1.0, Lucene HNSW graph search can run on existing Faiss indexes, allowing
partial byte loading and early termination. Memory-optimized search covers
Faiss binary indexes, and inner vector results can be rescored.

### Vector-search operations

Since 3.4.0, k-NN warmup supports memory-optimized search, including indexes
created before 2.18. SEISMIC sparse ANN supports nested fields during ingestion
and querying, and its query may omit `method_parameters`.

### Faiss efficient-filter exact-search control

Since 3.5.0, an index setting can disable the exact-search phase after ANN
search when Faiss efficient filters are used.

## Semantic fields and sparse retrieval

### Semantic and sparse neural search

Since 3.0.0, Neural Search provides semantic sentence highlighting with a
bundled question-answering model and custom tags, a semantic field mapper,
analyzer-based neural sparse queries, and a stats API.

### Semantic processing controls

Since 3.1.0, semantic fields can enable or disable chunking, use a
fixed-character-length chunker, and apply search analyzers at index creation
and query time. A neural sparse query cannot supply both a model ID and an
analyzer.

### Neural Search statistics

The 3.1.0 stats API adds `include_individual_nodes`, `include_all_nodes`, and
`include_info`, and covers more processors and algorithms. Invalid statistic
names now return a bad-request response instead of being ignored.

### Sparse retrieval and reranking

Since 3.3.0, Neural Search supports SEISMIC sparse approximate-nearest-neighbor
retrieval. k-NN and neural queries support native maximal marginal relevance,
the `lateInteractionScore` Painless function implements ColBERT-style
multi-vector rescoring, and vector-field creation accepts an optional top-level
`engine`.

### Semantic search processing

Since 3.3.0, the semantic-highlighting response processor can batch inference
requests to remote models, and semantic fields can use the sparse two-phase
processor.

### Neural and hybrid search

Since 3.5.0, Neural Search supports asymmetric embedding models, hybrid queries
over gRPC, and `min_score` on hybrid searches. SEISMIC sparse ANN queries can
participate in query explanation.

### Nested semantic highlighting

Since 3.7.0, request-level opt-in lets batch semantic highlighting process
nested-document `inner_hits`, returning the relevant nested passage rather than
only top-level content.
