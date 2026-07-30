# Search, Relevance, and Query Insights

Use this reference for hybrid-query construction, score fusion, aggregations,
relevance evaluation, Query Insights, and workload-aware search.

## Hybrid query construction and fusion

### Hybrid search controls and diagnostics

Since 2.19.0, hybrid queries accept `pagination_depth` for large result sets and
support reciprocal rank fusion (RRF) as a rank-based combination method. The
`hybrid_score_explanation` processor explains normalization and combination,
and `verbose_pipeline` exposes transformations across search-pipeline
processors.

### Deferred query templates and inference inputs

The 2.19.0 `template` query type leaves placeholders unresolved until a search
request processor assigns them. ML inference search request extensions let a
search supply additional model-specific input fields.

### Hybrid-query controls

Since 3.0.0, hybrid search supports Z-score normalization, a lower bound for
min-max normalization, and `inner_hits` for nested and parent-join results.
Filter functions are available to both hybrid and neural query builders.

### Hybrid query grouping and fusion

Since 3.1.0, hybrid queries support `collapse` for field-based grouping and
deduplication, and the RRF normalization processor accepts custom weights.
Invalid nested hybrid-query structures are rejected.

### Hybrid-query bounds and inner hits

Since 3.2.0, min-max normalization has an upper-bound parameter. A collapsed
hybrid query can return `inner_hits` for every group.

### Hybrid-query composition restriction

Since 3.6.0, a `hybrid` query is rejected inside compound queries such as
`function_score`, `constant_score`, or `script_score`.

## Aggregations and approximate queries

### Experimental aggregation and Discover features

OpenSearch 2.19.0 includes disabled-by-default star-tree aggregation for metric
aggregations and date histograms that contain metric aggregations. The same
release's experimental Discover view adds SQL and PPL alongside DQL and Lucene,
with autocomplete and improved data selection.

### Star-tree general availability

Star-tree indexes become production-ready in 3.1.0 for supported query and
aggregation workloads.

### Streaming aggregation

Since 3.2.0, segment-level partial aggregation results can stream to the
coordinating node instead of returning one response per shard, moving
high-cardinality reduction work away from data nodes.

### Approximate-query coverage

Since 3.2.0, approximate queries support `HALF_FLOAT`, `FLOAT`, `DOUBLE`,
`INTEGER`, `BYTE`, `SHORT`, and `UNSIGNED_LONG`, and can use `search_after`.

### Star-tree expansion and safeguards

Since 3.2.0, star-tree can accelerate aggregations whose queries target IP
fields. Index, node, and shard statistics report total, active, and elapsed-time
usage. Optimization is suppressed when DLS, FLS, or field masking applies, and
Custom Codecs supports composite indexes.

### Star-tree multi-terms aggregations

Since 3.3.0, star-tree indexes can accelerate `multi_terms` aggregations.
Index-, node-, and shard-level search statistics include star-tree failure
counts.

### Cardinality aggregation execution hints

OpenSearch 2.19.1 adds an execution hint for cardinality aggregations. Request
builders should only send it to versions that support it.

### Geospatial coordinate validation

Since 3.5.0, geospatial input validation enforces coordinate limits for lines,
polygons, and polygon holes.

## Learning to Rank and relevance evaluation

### Learning to Rank plugin

The 2.19.0 Learning to Rank plugin rescores results with lightweight models
such as XGBoost and RankLib. It uses `.ltrstore*` as a system index and supplies
settings, statistics, a circuit breaker, and read/full-access Security roles.

### Learning to Rank missing values

Since 3.2.0, Learning to Rank can evaluate XGBoost models whose feature inputs
contain missing values.

### Search Relevance Workbench

Introduced in 3.1.0, the workbench compares search algorithms and evaluates
quality from User Behavior Insights. It supports hybrid-search experiments,
external judgment imports, statistics, and ratings rather than scores for
judgments. Its backend root is `/_plugin/_search_relevance`.

### Search relevance workflow

In 3.2.0, the new workbench interface becomes the default with an opt-out.
Dashboards visualizes evaluation and hybrid-experiment results. Implicit
judgments can filter User Behavior Insights events by date, while hybrid
optimizer and pointwise experiments can run as scheduled tasks.

### Search relevance experiment controls

Since 3.4.0, Dashboards can schedule and deschedule experiments and compare
agentic search with single-query and pairwise tools. Experiment,
search-configuration, query-set, and judgment-list views support GUID filters.

### Search Relevance judgment and search APIs

Since 3.5.0, the workbench supports customizable prompt templates for
language-model judgments, and its comparison UI can reuse search
configurations. Search Configurations, Judgments, Query Sets, and Experiments
each expose a `_search` endpoint that accepts OpenSearch DSL.

### Search Relevance Workbench expansion

Since 3.6.0, the workbench supports multiple data sources and manual Query Set
creation from plain text, key-value, JSON Lines, or NDJSON. Evaluations add
Recall@K, mean reciprocal rank, and DCG@K. Precision, MAP, and other
binary-dependent metrics use dynamic percentile-based relevance thresholds.

### Experimental Relevance Agent

OpenSearch 3.6.0 adds a disabled-by-default Relevance Agent. Its multi-agent
Dashboards workflow analyzes user behavior, proposes relevance changes, and
validates them with offline evaluation.

### Search relevance imports and optimization

Since 3.7.0, Dashboards imports CSV judgment sets of up to 10,000 rows. The
hybrid optimizer supports Z-score and RRF across selected `rank_constant`
values, evaluates 82 variants per query, and lets an experiment opt into
particular techniques.

### Search Relevance Workbench general availability

OpenSearch 3.5.0 makes Search Relevance Workbench generally available rather
than experimental.

## Query Insights and live queries

### Query Insights operations

Since 2.19.0, Query Insights Dashboards provides historical top-N queries,
drill-downs, configuration, and retention. The backend supports fetch by ID and
automatic expiration. The custom local-index-name setting is removed.

### Live Query Insights

Since 3.0.0, Query Insights exposes an inflight/live-queries API for real-time
monitoring and a `verbose` option for top queries. Dashboards displays returned
columns dynamically.

### Query Insights controls

Since 3.1.0, Query Insights can exclude selected indexes, label historical
metrics, and report `isCancelled` in Live Queries. Dashboards has dedicated
Live Queries and Workload Management views.

### Query Insights limits and data sources

Since 3.2.0, Inflight Queries in Dashboards supports multiple data sources, and
the Query Insights reader search limit is 500.

### Query Insights data sources and settings

Since 3.4.0, Query Insights Dashboards uses version-aware settings and supports
multiple data sources on the Live Queries page. Its Workload Management view
can use security attributes.

### Query Insights identity and workload context

Since 3.5.0, top-N records can contain username and roles. Wrapper endpoints
around Query Insights settings support finer-grained access control, and the
Top-N dashboard integrates workload-group filtering and sorting.

### Query Insights access and live-query history

Since 3.6.0, Query Insights can filter by username and shared backend roles so
non-admin users see only authorized queries. Live Queries includes shard task
details, an on-demand cache of recently finished searches, and explicit tags
for failed queries.

### Query Insights recommendations and remote export

Since 3.6.0, an asynchronous rule service analyzes top-N queries and returns
recommendations with confidence and estimated impact. Top-N data can export as
timestamp-organized JSON to remote blob repositories, with S3 support.
Dashboards adds P90/P99 statistics plus distribution, line, and heatmap views.

### Query profiling and inline recommendations

Since 3.7.0, Query Insights provides a Dev Tools profiler with shard timing, a
collapsible query hierarchy, and navigation from Query Details. The Top Queries
API can include recommendations through `recommendations`.

## Workload-aware search

### Workload-management auto-tagging

Since 3.1.0, index-based rules can auto-tag requests with workload groups, so
tenant resource controls do not require every request to carry an explicit
header tag.

### Workload-aware query operations

Since 3.3.0, Live Queries can filter by workload group, and Dashboards links
workload groups and live queries bidirectionally. Auto-tagging rules also match
principal attributes such as username and role.

### Per-group workload settings

Since 3.7.0, a workload group can override search timeout, cancellation
interval, maximum bucket count, and other search settings for every request
routed to that group.
