# Upgrades and Core Compatibility

Use this reference for migration planning, runtime requirements, removed
settings and APIs, node roles, request limits, index compatibility, analyzers,
and core storage behavior.

## Migration blockers and data preparation

### Elasticsearch OSS migration

- Before migrating Elasticsearch OSS 6.8 data into OpenSearch 1.x, audit each
  document's total nested-object count. OpenSearch enforces the default
  `index.mapping.nested_objects.limit` of 10,000, which Elasticsearch 6.8 did
  not; an over-limit document can block shard relocation.

### Index compatibility

- OpenSearch 3.0 cannot use indexes created before 2.x, including system
  indexes. Reindex all of them before upgrading.
- OpenSearch 3.3.1 automatically sets `skip_list=true` for new `@timestamp`
  fields created since 3.3.0. Existing indexes with `@timestamp` or index-sort
  date fields retain `skip_list=false`; test both new-index and existing-index
  behavior.
- The 3.0 `romanian` analyzer normalizes cedilla forms to modern comma-based
  Unicode characters. Reindex existing Romanian text to keep old and new
  documents consistent.

### Request and stored-content limits

- In 3.0.0, bulk indexing enforces a 512-byte `_id` maximum and `_bulk` no
  longer accepts `batch_size`.
- OpenSearch 3.0 limits JSON object and array nesting to 1,000 levels and
  property names to 50,000 bytes or characters depending on the input source.
  Reshape over-limit requests and stored content before upgrade.
- Nested queries have a maximum depth in 3.0.0; exercise deeply generated
  queries in upgrade tests.
- Geospatial validation in 3.5.0 enforces coordinate limits for lines,
  polygons, and polygon holes.

## Runtime, packaging, and node roles

### Java and Lucene

- OpenSearch 3.0.0 requires JDK 21 and uses Lucene 10.1.0.
- The Java Security Manager is replaced in 3.0.0 by a Java agent that
  intercepts privileged calls. Policy files remain, with grants scoped to
  individual codebases and privileged actions.

### Artifact verification

- Artifacts from 3.0.0 onward use the `release@opensearch.org` PGP key, which
  expires March 6, 2027. The `opensearch@amazon.com` key is reserved for 2.x
  artifacts.

### Node roles and storage

- An empty role list configures a coordinating-only node in 3.0.0:

  ```yaml
  node.roles: []
  ```

- Searchable snapshots in 3.0 require the `warm` role on every node handling
  their shards; nodes with only the `search` role are insufficient.
- Remote-store-enabled clusters can use the 3.0.0 `_scale` API to turn off all
  index writers and make an index search-only, separating indexing and search
  capacity.

## Removed and renamed core configuration

- OpenSearch 3.0.0 removes `mmap.extensions` and the `transport-nio` plugin.
- OpenSearch 3.0 removes
  `compatibility.override_main_response_version`; the root response can no
  longer advertise a legacy version through that setting.
- OpenSearch 3.0 no longer exposes system indexes through the REST API.
- The camel-case tokenizer name `PathHierarchy` is deprecated in 3.0. Use
  `path_hierarchy` in new and updated analyzer configurations.
- Performance Analyzer RCA is replaced by Telemetry for 3.0. Gantt Charts is
  removed from the Dashboards bundle, and legacy Observability notebooks are
  unsupported.

## Corrected response and query behavior

### Nodes API formats

- In 3.0, `total_indexing_buffer_in_bytes` is a raw byte count such as
  `53687091`, while `total_indexing_buffer` is human-readable, such as
  `51.1mb`. Update consumers that assumed the opposite formats.

### Wildcard matching

- The 2.5 fix to `case_insensitive` wildcard queries on text fields removes
  matches that earlier releases returned incorrectly. Expect some upgraded
  queries and tests to return fewer results.

### Similarity default

- OpenSearch 3.0.0 changes the default scorer from
  `LegacyBM25Similarity` to `BM25Similarity`. Preserve an explicit similarity
  when the score change is not intended.

### Cardinality request compatibility

- OpenSearch 2.19.1 adds an execution hint for cardinality aggregations. Only
  send the option to versions that support it.

## SQL and Dashboards removals

- For 3.0, SQL deprecates its OpenSearch DSL format and several settings and
  removes the SparkSQL connector and `DELETE`.
- OpenSearch 3.0 removes `plugins.sql.pagination.api`, deprecates Scroll-based
  SQL pagination, and defaults to Point in Time. It also removes deprecated
  OpenDistro endpoints and `opendistro`-prefixed settings.
- OpenSearch Dashboards 3.0 removes `discover:newExperience` and the DataGrid
  table feature. Update saved workflows and deployment settings that depend on
  either.
- OpenSearch Dashboards 3.5.0 deprecates Node.js 20 while moving to Node.js 22
  and replaces Webpack 4 with the webpack-compatible Rspack bundler. No
  customer-facing API break was observed, but custom build integrations should
  be tested.
- The 2.19.0 platform notices announce upcoming removal of Ubuntu 20.04
  support for OpenSearch and Dashboards and Amazon Linux 2 support for
  Dashboards as its Node.js baseline moves beyond Node.js 18.

## Plugin and lifecycle compatibility

### Notifications dependencies

- Alerting notification actions on OpenSearch 2.0 require the
  `notifications-core` and `notifications` backend plugins. Managing them in
  Dashboards also requires `notificationsDashboards`.

### Workload terminology

- OpenSearch 3.0 renames query groups to workload groups:
  `wlm/query_group` becomes `wlm/workload_group`, response field
  `queryGroupID` becomes `workloadGroupID`, and settings use the
  `wlm.workload_group` prefix.

### Agent tool replacement

- OpenSearch 3.0 removes `CatIndexTool`. Use `ListIndexTool` in agent and tool
  configurations.

### Flow and replication lifecycle

- In 3.0.0, Dashboards Flow Framework expects ingestion input as JSON Lines.
- The 3.0.0 ISM `unfollow` action invokes stop replication for cross-cluster
  replication.

## Cryptographic compatibility

- The 3.0 Security plugin corrects Blake2b salt handling. The same inputs may
  hash differently than on earlier releases, so update integrations and test
  fixtures that compare generated hashes.
