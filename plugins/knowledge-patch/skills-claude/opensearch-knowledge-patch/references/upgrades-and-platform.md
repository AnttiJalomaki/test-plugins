# Upgrades and Platform Compatibility

Use this reference before changing major versions, runtimes, packaging,
analyzers, or clients that parse core responses.

## Migration blockers and required preparation

### Elasticsearch 6.8 migration nested-object limit

When migrating Elasticsearch OSS 6.8 data to OpenSearch 1.x, first audit every
document for the OpenSearch default `index.mapping.nested_objects.limit` of
10,000 nested JSON objects across all fields. Elasticsearch 6.8 did not enforce
that limit, and an oversized document can block shard relocation.

### Pre-2.x indexes block a 3.0 upgrade

OpenSearch 3.0 does not support indexes created before 2.x, including system
indexes. Reindex all of them before upgrading.

### Searchable snapshots require warm nodes

OpenSearch 3.0 no longer allows searchable snapshots on nodes with the `search`
role. Every node that handles their shards must have the `warm` role; update
roles before the upgrade.

### JSON parser limits

OpenSearch 3.0 limits JSON object and array nesting to 1,000 levels and property
names to 50,000 bytes or characters, depending on the input source. Reshape
requests and stored content that exceed either limit.

### OpenSearch 2.0 notification plugin dependencies

Alerting notification actions in 2.0 require the `notifications-core` and
`notifications` backend plugins. Dashboards management also requires
`notificationsDashboards`.

## Runtime, artifacts, and packaging

### Java and Lucene baseline

OpenSearch 3.0 requires JDK 21 as its minimum Java runtime and uses Lucene
10.1.0.

### Artifact-signing key

Artifacts for 3.0.0 and later use the `release@opensearch.org` PGP key, which
expires March 6, 2027. The `opensearch@amazon.com` key is reserved for 2.x
artifacts.

### Upcoming platform support deprecations

OpenSearch 2.19.0 announced the upcoming deprecation of Ubuntu 20.04 for
OpenSearch and Dashboards. It also announced deprecation of Amazon Linux 2 for
Dashboards as the Node.js baseline moves beyond Node.js 18.

### Dashboards runtime and bundler transition

OpenSearch Dashboards 3.5.0 deprecates Node.js 20 as it moves to Node.js 22. It
also replaces Webpack 4 with the webpack-compatible Rspack bundler; no
customer-facing API break was observed, but custom build integrations should be
tested.

### OpenSearch 3.0 migration notices

The 2.19.0 migration notices identify several 3.0 transitions:

- Telemetry replaces Performance Analyzer RCA.
- Gantt Charts leave the Dashboards bundle, and legacy Observability notebooks
  are unsupported.
- SQL deprecates its OpenSearch DSL format and several settings and removes the
  SparkSQL connector and `DELETE`.
- k-NN deprecates NMSLIB in favor of Faiss or Lucene.

## Removed and renamed core interfaces

### Core request and configuration breaks

Since 3.0.0, bulk indexing enforces a 512-byte `_id` limit and `_bulk` rejects
the deprecated `batch_size` parameter. The `mmap.extensions` setting and
`transport-nio` plugin are removed, and nested queries have a maximum depth.

### Removed main-response compatibility override

OpenSearch 3.0 removes
`compatibility.override_main_response_version`; a client can no longer use
that setting to make the root response advertise a legacy version.

### System-index REST access removal

OpenSearch 3.0 completes the earlier deprecation by no longer exposing system
indexes through the REST API.

### Nodes API indexing-buffer formats

Since 3.0.0, `total_indexing_buffer_in_bytes` is a raw byte count such as
`53687091`, while `total_indexing_buffer` is human-readable, such as `51.1mb`.
Update consumers that parse these fields.

### Coordinating-only nodes

Since 3.0.0, an empty role list explicitly configures a coordinating-only node:

```yaml
node.roles: []
```

### Query groups become workload groups

OpenSearch 3.0 renames query groups to workload groups:

- `wlm/query_group` becomes `wlm/workload_group`.
- Responses use `workloadGroupID` instead of `queryGroupID`.
- Related cluster settings use the `wlm.workload_group` prefix.

### ML Commons index-listing tool replacement

OpenSearch 3.0 removes `CatIndexTool`. Update agent and tool configurations to
use `ListIndexTool`.

### Discover removals

OpenSearch Dashboards 3.0 removes the `discover:newExperience` setting and the
DataGrid table feature. Adjust deployments and saved workflows that depend on
either.

## Analyzer, query, and cryptographic corrections

### Wildcard query case handling

The 2.5 correction to `case_insensitive` wildcard queries on text fields can
remove matches that earlier versions returned incorrectly. Expect some queries
and tests to return smaller result sets after upgrade.

### Path hierarchy tokenizer name

The camel-case `PathHierarchy` tokenizer name is deprecated in 3.0. Use
`path_hierarchy` in new and updated analyzer configurations.

### Blake2b salt behavior

The 3.0 Security plugin corrects Blake2b salt handling, so identical inputs can
produce hashes different from earlier releases. Update integrations and tests
that compare generated Blake2b values.

### Romanian analyzer normalization

The 3.0 `romanian` analyzer normalizes cedilla forms to modern comma-based
Unicode characters. Reindex existing Romanian text so old and new documents
analyze consistently.

### Date-field skip-list compatibility

OpenSearch 3.3.1 automatically sets `skip_list=true` for new `@timestamp`
fields created since 3.3.0. Existing indexes with `@timestamp` or index-sort
date fields retain `skip_list=false`; test new-index and existing-index upgrade
paths separately.

## Security and sandbox migration

### Java-agent security sandbox

OpenSearch 3.0 replaces the Java Security Manager with a Java agent that
intercepts privileged calls. The policy-file model remains: grants attach to
individual codebases for the privileged actions they may perform.

### Security configuration changes

Since 3.0.0, the Security plugin removes its OpenSSL provider and renames
whitelist settings to allowlist settings. The `_cat/shards` action requires
`cluster:monitor/shards`, `ignore_hosts` accepts CIDR ranges, and
password-strength validation accepts `good`.
