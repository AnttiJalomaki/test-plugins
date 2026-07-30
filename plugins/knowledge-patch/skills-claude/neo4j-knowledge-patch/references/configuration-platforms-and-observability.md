# Configuration, Platforms, APIs, and Observability

Use this reference for packaged configuration behavior, log and metric
contracts, HTTP clients, CDC consumers, Java notification integrations, and
edition-specific packaging.

## Packaged Cypher default

Starting in 2026.02, distributed `neo4j.conf` files explicitly contain:

```properties
db.query.default_language=CYPHER_25
```

New deployments that use the packaged configuration therefore default newly
created databases to Cypher 25. Existing databases and deployments with
preserved configuration need not inherit this setting; inspect actual state.

## Query log format

For new installations, the release after the next LTS will default query logs
to JSON rather than PLAIN. Upgrades that retain `server-logs.xml` keep their
existing format, and PLAIN remains supported. JSON records more information but
uses more storage. Capacity-plan before changing formats and make log parsers
accept the selected schema.

The JSON query log's `failureReason` column is deprecated from 2025.05. Use
`errorInfo` in log schemas, mappings, and field lookups.

## Debug and configuration logs

The default `debug.log` format changes from text to JSON. Keep the default
appender because it is the supported diagnostic format; add another appender
when a second representation is necessary.

On a new installation, or an upgrade that replaces configuration, these paths
also change:

```text
server.logs.config: conf/server-logs.xml -> server-logs.xml
server.logs.user.config: conf/user-logs.xml -> user-logs.xml
```

Relative values resolve from `server.directories.configuration`, no longer
from `server.directories.neo4j_home`.

## Default metric selection

`cluster.internal.*` is no longer present in the defaults of the metrics
settings. Those internal metrics were not intended for customer use. Configure
supported metrics explicitly and do not rely on implicit collection of the
internal family.

From 2025.03, default `server.metrics.filter` includes the `neo4j.count`
metrics class instead of deprecated `ids_in_use`. Monitoring that inherits the
default must consume the data-count metrics rather than expecting the old
class.

## Raft and store metrics

The HTTP status field `raftCommandsPerSecond` is deprecated. Monitor
`<prefix>.cluster.raft.commit_index` on every server and alert on divergence
between values.

These Raft series are also retiring:

- `<prefix>.cluster.raft.in_flight_cache.max_bytes` and
  `<prefix>.cluster.raft.in_flight_cache.max_elements` are deprecated from
  2026.07 and will be removed after the next LTS.
- `<prefix>.cluster.raft.tx_retries` has been deprecated since 2025.02 and will
  be removed in a future release.

Separately, upgrades must replace the removed `causal_clustering.core` Raft
metrics, the moved `causal_clustering.read_replica.pull_update*` series, and
`<prefix>.store.size.total`; see the upgrade reference for the complete map.

## HTTP Query API

### Transaction IDs

Query API transaction identifiers are six characters long rather than four
(since 2026.04.0). Integrations must not enforce the old length in validation,
database columns, caches, or UI fields.

### Transactional endpoint migration

The transactional HTTP API is deprecated in 5.26 in favor of the HTTP Query
API. Query API endpoints are enabled by default from 5.26. On earlier releases,
enable them by adding `QUERY_API_ENDPOINTS` to:

```properties
server.http_enabled_modules
```

Migrate request construction, transaction handling, authentication, error
handling, and response parsing before the transactional API is removed.

### GQLSTATUS errors

Programmatic use of error-message text is deprecated from 2025.04 because the
wording can change. Parse GQLSTATUS error codes and branch on those stable codes
instead. Cypher Shell similarly defaults `--error-format` to `gql`; scripts
that need another format must request it explicitly.

## Java notification APIs

The server-side Notification API and the Result Core API's
`getNotifications()` method are deprecated from 5.26. Java integrations should
stop relying on those notification entry points and move to the supported
diagnostic/status contract used by their driver or API.

## Change Data Capture

`db.cdc.current()` returns `txCommitTime` as of 2026.06.0. A CDC client can
retrieve the commit time of its most recent transaction alongside the
transaction identifier.

Migrate the beta procedure namespace:

```text
cdc.current() -> db.cdc.current()
cdc.earliest() -> db.cdc.earliest()
cdc.query() -> db.cdc.query()
```

Update procedure allowlists, generated calls, row mappings, and checkpoints.

## Query-plan version strings

`EXPLAIN` and `PROFILE` consistently report the underlying Neo4j version down
to the point release. Tools that parse, snapshot, or compare plans must expect
the additional version detail.

## Browser and fleet packaging

From 2026.02.3, Community Edition contains a `web/` directory with Neo4j Browser
as a ZIP file. Enterprise Edition has no `web/` directory and continues to ship
Browser as a JAR under `lib/`. Packaging scripts must branch on edition rather
than assuming the same artifact location.

Enterprise Fleet Management is no longer a separately bundled DBMS package,
because its functionality is included in Neo4j. Local-network discovery and
bulk registration are described in the administration reference.

## Observability migration checks

1. Preserve or intentionally replace log configuration, then validate JSON
   parsing and storage capacity.
2. Replace query-log `failureReason` with `errorInfo`.
3. Inventory dashboards for internal, ID-use, old Raft, read-replica, and
   renamed store-size metrics.
4. Expand Query API transaction-ID storage and migrate transactional HTTP
   clients.
5. Branch application logic on GQLSTATUS codes and migrate Java notification
   consumers.
6. Update CDC namespace calls, row mappings, and checkpoint data for commit
   timestamps.
