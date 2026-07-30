# Vector, Hybrid, and Search Relevance

Use this reference for k-NN mappings and engines, compressed vectors, remote
and GPU builds, hybrid and semantic queries, sparse retrieval, star-tree
indexes, Learning to Rank, and Search Relevance Workbench.

## k-NN mappings, source, and retrieval

### Mapping validation and index settings

- Since 2.19.0, a training-backed vector mapping cannot specify both a model ID
  and `dimension`; the model's training index supplies the dimension.
- `index.knn` is immutable after index creation (2.19.0).
- Derived vector source cannot be enabled when `index.knn=false` (3.1.0).
- Mode and compression settings are rejected for indexes created before
  2.17.0 (3.1.0).
- Vector-field creation accepts an optional top-level `engine` (3.3.0).

### Derived vector source

- The experimental 2.19.0 derived-source mode removes k-NN vectors from stored
  JSON `_source` and injects them when a document is read. It supports flat
  mappings, object fields, and one nested level and is disabled by default.
- Derived vector source is production-ready in 3.0.0 across Faiss, Lucene, and
  NMSLIB.

### Returning vectors

- Searches using `fields` work with `knn_vector` from 2.19.0.
- In 3.7.0, `docvalue_fields` can retrieve float, byte, and binary
  `knn_vector` values from Lucene and Faiss indexes at every compression level
  without reindexing. The default representation is base64-encoded binary
  rather than an array.

### Nested vectors and highlights

- Nested k-NN `inner_hits` can return multiple values with Lucene or Faiss
  (2.19.0). NMSLIB adds `expand_nested_docs`, and neural k-NN adds
  `expand_nested`.
- From 2.19, map nested `text_embedding` inputs directly instead of using
  nested `_ingest._value` substitution:

  ```json
  "field_map": {
    "books.title": "title_embedding"
  }
  ```

- SEISMIC sparse ANN supports nested fields during ingestion and querying in
  3.4.0, and its query may omit `method_parameters`.
- In 3.7.0, batch semantic highlighting can process nested-document
  `inner_hits` through a request-level opt-in and return the relevant nested
  passage instead of only top-level content.

## Engines, execution, and compression

### Engine compatibility

- Lucene supports binary vector indexes from 2.19.0.
- Faiss supports cosine similarity and radial search without caller-side
  normalization from 2.19.0.
- The implicit k-NN engine changed from NMSLIB to Faiss in 2.18. With
  `space_type: "cosinesimil"` and no explicit engine, indexing normalizes
  vectors, so stored values differ from inputs. Already-normalized vectors can
  use `innerproduct` for equivalent scoring without implicit normalization.
- NMSLIB is deprecated in favor of Faiss or Lucene as part of the 3.0
  migration.

### Search execution

- Concurrent segment search is enabled by default for k-NN in 3.0.0.
- Faiss explain support in 3.0.0 covers exact, ANN, radial, and disk-based
  searches.
- `memory-optimized-search` provides a lower-memory Faiss mode in 3.0.0.
- Lucene HNSW graph search can execute directly over Faiss indexes in 3.1.0,
  enabling partial byte loading and early termination.
- Memory-optimized search supports Faiss binary index types in 3.1.0, and
  inner vector search results can be rescored.
- k-NN warmup supports memory-optimized search in 3.4.0, including indexes
  created before 2.18.
- A 3.5.0 index setting can disable the exact-search phase after ANN when
  Faiss efficient filters are used.

### Rescoring and failure behavior

- `rescore: false` actually disables rescoring from 2.19.0.
- New OnDisk indexes with 4x compression rescore by default in 3.1.0. Set
  `rescore: false` to preserve the prior behavior.
- A terminal remote vector-index-build failure no longer falls back to CPU in
  3.2.0.

### Memory and circuit breakers

- k-NN adds node-level circuit breakers for heterogeneous memory limits in
  3.0.0.
- Legacy index-level settings for `ef_construction`, `m`, space type, and
  plugin enablement are removed in 3.0.0.

### Quantization and recall

- Binary-quantized Faiss indexes in 3.2.0 can use asymmetric distance
  computation, comparing a full-precision query vector with compressed
  document vectors, and random rotation to reduce loss at 32x compression.
  Asymmetric distance also works when Lucene graph search executes on Faiss.
- Lucene BBQ and Faiss scalar quantization support 1-bit vectors at 32x
  compression in 3.6.0 for approximate and exact search, including Lucene
  flat format and Faiss memory-optimized search.
- Faiss 32x compression defaults to the SQ 1-bit encoder in 3.6.0.
- Vector metadata can use Zstandard compression in 3.6.0, and byte vectors gain
  a Hamming-distance scorer.
- Remote index builds support 1-bit scalar quantization in 3.7.0.

## Vector index building

- GPU vector operations are a disabled-by-default experiment in 3.0.0.
- GPU-accelerated index building becomes production-ready in 3.1.0.
- Remote vector index building is enabled by default in 3.1.0 through
  `index.knn.remote_index_build.enabled`.
- GPU index builds support FP16, byte, and binary vectors in addition to FP32
  in 3.2.0.

## Hybrid search

### Fusion, pagination, and diagnostics

- Hybrid queries add `pagination_depth` for large result sets and reciprocal
  rank fusion (RRF) in 2.19.0.
- Use `hybrid_score_explanation` to explain normalization and combination and
  `verbose_pipeline` to expose transformations across search-pipeline
  processors (2.19.0).
- Hybrid search adds Z-score normalization and a lower bound for min-max
  normalization in 3.0.0.
- The RRF normalization processor accepts custom weights in 3.1.0.
- Min-max normalization adds an upper bound in 3.2.0.
- The 3.7.0 hybrid optimizer supports Z-score normalization and RRF across
  selected `rank_constant` values. It evaluates 82 variants per query and can
  limit an experiment to selected techniques.

### Filtering, grouping, and nesting

- Filter functions are available to hybrid and neural query builders in
  3.0.0.
- Hybrid queries support `inner_hits` for nested and parent-join results in
  3.0.0.
- Hybrid queries support `collapse` for field grouping and deduplication in
  3.1.0.
- Invalid nested hybrid-query structures are rejected in 3.1.0.
- A collapsed hybrid query can return `inner_hits` for each group in 3.2.0.
- Hybrid searches support `min_score` in 3.5.0 and hybrid queries can run over
  gRPC.
- From 3.6.0, a `hybrid` query is rejected inside compound queries such as
  `function_score`, `constant_score`, and `script_score`.

## Semantic and sparse neural search

### Semantic fields and highlighting

- Neural Search in 3.0.0 adds semantic sentence highlighting with a bundled QA
  model and custom tags, a semantic field mapper, analyzer-based neural sparse
  queries, and a stats API.
- Semantic fields in 3.1.0 can toggle chunking, use fixed-character-length
  chunks, and apply search analyzers at index creation and query time.
- A neural sparse query cannot supply both a model ID and an analyzer
  (3.1.0).
- The Neural Search stats API in 3.1.0 adds `include_individual_nodes`,
  `include_all_nodes`, and `include_info`, covers more processors and
  algorithms, and rejects invalid statistic names with a bad request.
- Semantic fields in 3.2.0 can configure generated dense `knn_vector` engine,
  mode, compression level, and method. They also add ingest batch sizing,
  sparse prune strategies, configurable chunking, embedding reuse, and
  `TOKEN_ID` sparse embeddings.
- The semantic-highlight response processor can batch remote inference in
  3.3.0. Semantic fields can use the sparse two-phase processor.
- Neural Search supports asymmetric embedding models in 3.5.0.

### Sparse retrieval and reranking

- Neural Search adds SEISMIC sparse approximate-nearest-neighbor retrieval in
  3.3.0.
- k-NN and neural queries gain native maximal marginal relevance in 3.3.0.
- The `lateInteractionScore` Painless function provides ColBERT-style
  multi-vector rescoring in 3.3.0.
- SEISMIC sparse ANN participates in query explanation in 3.5.0.

## Approximate queries, aggregation, and star-tree

### Approximate query coverage

- Approximate queries support `HALF_FLOAT`, `FLOAT`, `DOUBLE`, `INTEGER`,
  `BYTE`, `SHORT`, and `UNSIGNED_LONG` and can use `search_after` in 3.2.0.

### Streaming aggregation

- In 3.2.0, shards can stream segment-level partial aggregation results to the
  coordinating node, moving high-cardinality reduction work away from data
  nodes.

### Star-tree lifecycle and safeguards

- Experimental star-tree aggregation in 2.19.0 supports metric aggregations
  and date histograms containing metric aggregations.
- Star-tree indexes become production-ready in 3.1.0.
- Star-tree accelerates aggregations over IP-field queries in 3.2.0. Index,
  node, and shard statistics report total, active, and elapsed-time usage.
- Star-tree optimization is suppressed when DLS, FLS, or field masking applies
  (3.2.0).
- Custom Codecs adds composite-index support in 3.2.0.
- Star-tree accelerates `multi_terms` aggregations in 3.3.0, and search
  statistics at index, node, and shard scope add failure counts.
- The AdditionalCodecs registration path in 3.5.0 lets plugins such as k-NN,
  Neural Search, and Security Analytics use custom codecs.
- OpenSearch Custom Codecs adds Intel QAT-accelerated Zstandard compression in
  3.1.0.

## Learning to Rank

- The Learning to Rank plugin introduced in 2.19.0 rescores with lightweight
  models such as XGBoost and RankLib. It uses `.ltrstore*` as a system index
  and includes settings, statistics, a circuit breaker, and read/full-access
  security roles.
- Learning to Rank can evaluate XGBoost models with missing input features in
  3.2.0.

## Search Relevance Workbench

### Data and evaluation

- The 3.1.0 workbench compares algorithms and evaluates quality using User
  Behavior Insights, hybrid experiments, and imported judgments. Its backend
  root is `/_plugin/_search_relevance`, it exposes statistics, and judgments
  are ratings rather than scores.
- In 3.2.0 its redesigned interface becomes the default with an opt-out.
  Dashboards visualizes evaluation and hybrid-experiment results; implicit
  judgments can filter User Behavior Insights by date, and hybrid-optimizer
  and pointwise experiments can run as scheduled tasks.
- In 3.4.0, experiments can be scheduled and descheduled in the UI, agentic
  search can be compared in single-query and pairwise tools, and GUID filters
  are available for experiments, search configurations, query sets, and
  judgment lists.
- Search Relevance Workbench is generally available in 3.5.0.
- In 3.5.0 it adds customizable judgment prompt templates, reusable comparison
  configurations, and OpenSearch DSL `_search` endpoints for Search
  Configurations, Judgments, Query Sets, and Experiments.

### Expanded metrics and optimization

- In 3.6.0 the workbench supports multiple data sources and manual Query Set
  creation from plain text, key-value, JSON Lines, or NDJSON.
- Evaluations in 3.6.0 add Recall@K, mean reciprocal rank, and DCG@K.
  Precision and MAP use dynamic percentile-based relevance thresholds.
- The disabled-by-default 3.6.0 Relevance Agent uses a multi-agent Dashboards
  workflow to analyze user behavior, propose changes, and validate them with
  offline evaluation.
- In 3.7.0 Dashboards imports CSV judgment sets of up to 10,000 rows directly.
