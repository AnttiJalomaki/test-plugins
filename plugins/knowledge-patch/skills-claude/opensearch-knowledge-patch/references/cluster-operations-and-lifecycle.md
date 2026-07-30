# Cluster Operations and Data Lifecycle

Use this reference for remote-store topology, ingestion and transport, Index
State Management, replication, scheduler APIs, and remote plugin metadata.

## Remote store and reader/writer separation

### Search-only remote-store indexes

Since 3.0.0, remote-store-enabled clusters can separate indexing and search
traffic. The `_scale` API can turn off every writer and make an index
search-only, allowing indexing and search capacity to scale independently.

## Transport and ingestion

### Experimental transport and ingestion

OpenSearch 3.0.0 includes disabled-by-default protobuf-over-gRPC transport,
pull-based ingestion from Apache Kafka and Amazon Kinesis with native
backpressure, and GPU acceleration for vector operations.

### gRPC transport reaches general availability

In 3.2.0, protobuf-over-gRPC becomes production-ready for bulk ingestion, with
expanded search coverage, k-NN queries, and encryption in transit.

### gRPC and Arrow transport coverage

Since 3.3.0, production gRPC supports term-level, full-text, geographic,
Boolean, script, and nested queries, and OpenSearch protobuf Python packages
are published to PyPI. A separate disabled-by-default Apache Arrow Flight
transport provides secured server-side node-to-node streaming through
`StreamTransportService`.

### Expanded gRPC requests

Since 3.4.0, gRPC search supports `ConstantScoreQuery`, `FuzzyQuery`,
`MatchBoolPrefixQuery`, `MatchPhrasePrefix`, `PrefixQuery`, and `MatchQuery`.
Bulk gRPC accepts documents encoded as CBOR, SMILE, or YAML.

### Pull-based ingestion general availability

OpenSearch 3.6.0 makes pull-based ingestion generally available and adds warmup
settings and adaptive shard selection.

## Index State Management and rollups

### Index State Management transition conditions

Since 3.2.0, ISM transitions support `no_alias` and `min_state_age`.

### Index State Management exclusions

Since 3.4.0, ISM index patterns can contain exclusion patterns.

### Index State Management and rollups

Since 3.5.0, `convert_index_to_remote` accepts optional `rename_pattern`, and
the `search_only` action supports reader/writer separation. Rollups add
cardinality metrics and multi-tier rollup support.

### Index policy simulation and analyzer reloads

Since 3.7.0, ISM Simulate evaluates every policy transition against live index
metrics and reports the next state without changing cluster state.
`_refresh_search_analyzers` accepts `reload_cached_resources` to hot-reload
resources such as Hunspell dictionaries and works on metadata-write-blocked
indexes such as CCR followers.

## Cross-cluster replication

### Cross-cluster replication controls

Since 3.7.0, every replication REST API accepts `cluster_manager_timeout`.
Stop, pause, start, and resume can clear stale persistent tasks. Replication
leaves `number_of_replicas` unchanged when a follower uses
`auto_expand_replicas`.

## Scheduler and snapshots

### Second-level schedules and scheduler APIs

Since 3.2.0, `IntervalSchedule` accepts seconds as a unit. Job Scheduler has
REST APIs to list jobs, optionally by node, list all locks, and retrieve one
lock.

### Snapshot and scheduler operations

Since 3.3.0, Snapshot Management can delete manually created snapshots, and Job
Scheduler provides a Job History Service.

## Remote plugin metadata

### Remote metadata mutation controls

Since 3.3.0, the Remote Metadata SDK supports global resources and
sequence-number/primary-term concurrency controls on put and delete. Put,
update, delete, and bulk operations accept refresh policies and timeouts.

### Remote metadata encryption

Since 3.4.0, the Remote Metadata SDK can encrypt and decrypt customer data with
customer-managed keys and can assume a role for those key operations.
