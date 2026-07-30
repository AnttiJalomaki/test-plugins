# PPL, SQL, Query APIs, and Transports

Use this reference for Calcite routing and semantics, PPL commands and
functions, SQL compatibility, unified query APIs, gRPC and Arrow request
coverage, and pull-based ingestion.

## PPL engine routing and request behavior

### Calcite defaults and fallback

- Calcite is enabled by default for PPL in 3.0.0.
- Failed Calcite PPL queries no longer fall back to the v2 engine by default
  in 3.2.0.
- Unsupported Calcite PPL commands do implicitly fall back to v2 in 3.3.0;
  distinguish unsupported syntax from a generally failed query.
- When `plugins.ppl.syntax.legacy.preferred=false`, `join` defaults to `max=1`
  in 3.5.0.

### Limits, cancellation, and diagnostics

- From 3.1.0, `query.size_limit` limits only final results, not intermediate
  processing.
- Calcite queries support timeouts in 3.4.0. Subsearch and join limits are
  configurable, and zero or negative `subsearch.maxout` means unlimited.
- Explain can return YAML in 3.4.0.
- PPL profiling reports phase-level and operator-level metrics in 3.5.0.
- PPL queries can be cancelled through `_tasks/_cancel` in 3.6.0 and accept
  `fetch_size`.
- The 3.6.0 grammar-bundle API supports third-party query tooling.

### Result and error semantics

- `count(*)` and `dc` cap at `MAX_INTEGER` in 3.3.0.
- Decimal `mod` returns a decimal in 3.3.0.
- An unmatched index pattern raises `IndexNotFoundException` in 3.3.0.
- In 3.6.0, final struct values are maps rather than lists.
- Missing or null `JSON_EXTRACT` paths return null in 3.6.0, as does arithmetic
  that overflows a double to infinity.
- `FIRST`, `LAST`, and `TAKE` accept text fields and scripts in 3.6.0.
- `NOT IN` and `NOT LIKE` exclude null or missing values in 3.7.0.
- `COALESCE(null, integer)` retains an integer type in 3.7.0.
- PPL wildcard searches in 3.7.0 no longer silently drop documents when a
  field is text in one index and keyword in another.
- Calcite `dedup` preserves sort order in 3.7.0.
- Dotted-path `eval` assignments preserve their map root in 3.7.0.
- `json_set` and `json_delete` handle `$.key` paths in 3.7.0.

## PPL data access and pipeline commands

### Metadata, schemas, and data sources

- PPL can reference metadata fields from 2.19.0.
- The 3.1.0 Calcite path supports `match_only_text` fields and merges object
  fields during index-schema discovery.
- Direct-query data sources are supported in 3.3.0.
- PIT searches use `_shard_doc` as the default sort tiebreaker in 3.3.0.

### Relational and pipeline operations

- The 3.0.0 Calcite PPL path supports `join`, `lookup`, `IN`, relation,
  `exists`, and scalar subqueries.
- It also supports `BETWEEN`, `dedup`, `parse`, JSON casts, `CASE`, `TYPEOF`,
  multiple index patterns, nested fields, comments, and additional scalar and
  data types.
- PPL 3.1.0 adds `flatten`, `expand`, `trendline`, `appendcol`, `grok`, `top`,
  `rare`, `fillnull`, `describe`, and `eventstats`.
- PPL 3.3.0 adds `spath`, `rex`, and `regex`; `rex` supports `sed` mode and
  `offset_field`.
- PPL 3.3.0 also adds `append`, expands `join` with field lists and options,
  adds time modifiers to `search`, supports wildcard `rename`, and implements
  Calcite `timechart`.
- Calcite PPL 3.4.0 adds `chart`, `streamstats`, `multisearch`, `replace`, and
  `appendpipe`.
- PPL 3.5.0 adds `addtotals`, `addcoltotals`, `transpose`, and `mvcombine`.
- PPL 3.6.0 adds bidirectional `graphlookup` with literal starting values,
  `convert` with five conversion functions, `mvexpand`, `nomv`, and
  `fieldformat`.
- PPL 3.6.0 also adds result highlighting, `spath` auto-extraction, a
  `contains` operator, and trailing or empty pipes.
- PPL 3.7.0 adds `union` with type coercion and UNION ALL semantics.
- Predicates accept `IS [NOT] NULL` in 3.7.0, and `head` and `top` accept
  `limit=N`.

### Extraction and conversion behavior

- `rex`, `spath`, and `parse` extractions perform automatic type conversion in
  3.4.0.
- `spath` supports dynamic fields in 3.5.0.
- `lookup` accepts `OUTPUT` as an alias for `REPLACE` in 3.5.0.
- PPL 3.7.0 extends `convert` with `ctime`, `mktime`, `mstime`, `dur2sec`, and
  a `timeformat` parameter.

## PPL functions and expressions

### General functions

- PPL 3.1.0 adds `DISTINCT_COUNT_APPROX`, `earliest`, `latest`, `coalesce`,
  `isempty`, `isblank`, `ispresent`, `geoip`, `cidrmatch`, JSON functions,
  cryptographic hashes, lambdas, array functions, and decimal literals.
- Calcite 3.2.0 adds `compare_ip`, IP casts, function-argument coercion,
  improved date comparisons, and broader expression support.
- Date/time functions default to UTC across PPL and SQL in 3.2.0.
- PPL 3.3.0 adds `values` and `list` statistics, `first` and `last`
  aggregates, `mvjoin`, `strftime`, `regex_match`, and nonnumeric and
  eval-context `max`/`min`.
- PPL 3.3.0 accepts ISO 8601 strings and adds `distinct_count`, `earliest`, and
  `latest` to `eventstats`.
- `geoip` accepts IP-typed input in 3.4.0.
- Eval division returns a decimal in 3.4.0.
- `like` accepts an optional `case_sensitive` argument in 3.4.0.

### Multivalue and replacement functions

- PPL 3.4.0 adds `mvdedup`, `mvindex`, and `mvappend`.
- `replace` supports wildcards in 3.4.0, its eval form supports regex, and
  `regexp_replace()` is an alias.
- PPL 3.5.0 adds `tonumber`, `mvzip`, `split`, `mvfind`, and `mvmap`.
- The 3.5.0 `ml` command accepts `category_field`.

### Aggregation and time controls

- `timechart` in 3.4.0 adds per-second, per-minute, per-hour, and per-day
  functions and can select its timestamp with `timefield`.
- `streamstats` and `eventstats` add `bucket_nullable` in 3.4.0.
- `top` and `rare` add `usenull` in 3.4.0.
- Span expressions support milliseconds, decimal literals, and an implicit
  `@timestamp` in 3.4.0.

## SQL and unified query APIs

### SQL compatibility

- SQL cursor continuation in 3.7.0 remains within the original query indexes
  under fine-grained access control.
- SQL 3.7.0 adds the `vectorSearch()` table function with k-NN pushdown and
  filtering modes.

### Unified query paths

- SQL 3.6.0 adds a unified query parser API, unified-API profiling, and native
  Calcite SQL planning.
- The query-only unified V2 path in 3.7.0 blocks DML and DDL and supports
  joins, `IN` and `EXISTS` subqueries, derived tables, window functions,
  `LIMIT`/`OFFSET`, and `LENGTH`, `REGEXP_REPLACE`, and `DATE_TRUNC`.

## PPL Alerting integration

- PPL Alerting 3.4.0 adds monitor execution and statistics, get/search/delete
  monitor operations, and alert retrieval and lifecycle operations.
- Alerting V2 roles are added to `roles.yml` in 3.4.0.
- Dashboards bucket-level monitor triggers can carry keyword filters in
  3.4.0.
- PPL can create or update Prometheus rules in 3.6.0.
- Experimental PPL Alerting assets are removed in 3.6.0 pending refactoring.
  Dashboards PPL Alerting moves to v1 endpoints and no longer maintains
  separate legacy and PPL paths.

## gRPC, Arrow, HTTP/3, and ingestion

### gRPC lifecycle and protection

- Protobuf-over-gRPC is a disabled-by-default experiment in 3.0.0.
- In 3.2.0 it becomes production-ready for bulk ingestion, expands search and
  k-NN coverage, and supports encryption in transit.
- The 3.3.0 production transport adds term-level, full-text, geographic,
  Boolean, script, and nested queries.
- OpenSearch protobuf Python packages are published to PyPI in 3.3.0.
- gRPC search in 3.4.0 adds `ConstantScoreQuery`, `FuzzyQuery`,
  `MatchBoolPrefixQuery`, `MatchPhrasePrefix`, `PrefixQuery`, and `MatchQuery`.
- Bulk gRPC requests in 3.4.0 can carry CBOR, SMILE, or YAML documents.
- Hybrid queries run over gRPC in 3.5.0.

### Other transports

- The disabled-by-default Apache Arrow Flight transport in 3.3.0 provides
  secured server-side streaming for node-to-node communication through
  `StreamTransportService`.
- Server-side HTTP/3 is a disabled-by-default experiment in 3.5.0.

### Pull-based ingestion

- Pull-based ingestion from Apache Kafka and Amazon Kinesis is a
  disabled-by-default 3.0.0 experiment with native backpressure.
- Pull-based ingestion becomes generally available in 3.6.0 and adds warmup
  settings and adaptive shard selection.

## Dashboards query experience

- The disabled-by-default experimental Discover view in 2.19.0 adds SQL and
  PPL alongside DQL and Lucene, with autocomplete and improved data selection.
