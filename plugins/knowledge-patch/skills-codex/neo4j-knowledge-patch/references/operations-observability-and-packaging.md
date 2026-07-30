# Operations, Observability, and Packaging

## Release and page-cache safety

The base 2025.06 release can sporadically deadlock on the checkpoint mutex.
Production deployments should use `2025.06.1` or later.

Linux deployments can opt into initial `io_uring` support for the background
page evictor and checkpointer (2026.04.0):

```properties
server.memory.pagecache.async=true
```

## Metrics defaults and migrations

`cluster.internal.*` is no longer included in default metric-setting values
(2025.06). These metrics were not intended for customer use; do not rely on
implicit collection.

From 2025.03, the default `server.metrics.filter` includes `neo4j.count` in
place of the deprecated `ids_in_use` class. Update monitoring that relies on
the default filter.

The old `causal_clustering.core` Raft metrics for indexes, term, leadership,
retries, in-flight cache, prefetch buffering, message processing, replication,
and last-leader messages are removed in favor of the Raft metrics. The three
`causal_clustering.read_replica.pull_update*` metrics move to store-copy
metrics. Six discovery-v1 metrics under `cluster.discovery` are removed
without replacements.

`<prefix>.store.size.total` is renamed to `<prefix>.store.size.full`; update
dashboards and alerts.

The HTTP status field `raftCommandsPerSecond` is deprecated. Monitor
`<prefix>.cluster.raft.commit_index` on every server and look for divergence.

`<prefix>.cluster.raft.in_flight_cache.max_bytes` and
`<prefix>.cluster.raft.in_flight_cache.max_elements` are deprecated from
2026.07 and will be removed after the next LTS.
`<prefix>.cluster.raft.tx_retries`, deprecated since 2025.02, will also be
removed.

## Log formats and locations

For new installations, the release after the next LTS will default query logs
to JSON rather than PLAIN (2026.05.0). An upgrade that retains
`server-logs.xml` keeps its current format. PLAIN remains supported; JSON
contains more information but produces larger logs.

The JSON query log's `failureReason` field is deprecated from 2025.05. Migrate
parsers and schemas to `errorInfo`.

The default `debug.log` format changes from text to JSON. Keep the default
appender for supportability; add another appender if a second format is
required. Parsers of the default file must accept JSON.

Import-progress paths change twice. In 2026.03, progress moves from
`server/logs/neo4j-admin-import-yyyy-MM-dd.HH.mm.ss.log` to
`server/data/imports/dbname-yyyy-MM-dd.HH.mm.ss/import.log`. In 2026.04, the
generated import-information directory moves back under `server/logs/`.

For new installations or upgrades that replace configuration, query
annotation formatting defaults to JSON and metrics CSV rotation compression
defaults to ZIP. `server.logs.config` and `server.logs.user.config` change
from `conf/server-logs.xml` and `conf/user-logs.xml` to `server-logs.xml` and
`user-logs.xml`; relative paths resolve from `server.directories.configuration`
rather than `server.directories.neo4j_home`.

## Fleet and deployment discovery

Self-managed Enterprise deployments registered with Fleet Manager can opt in
to collection of security logs for Aura Console Security Log Analyzer
(2026.04.0). Registration alone does not enable log collection.

Enterprise Fleet Management is no longer bundled separately with the DBMS
package because the functionality is included in Neo4j (2026.04.0).
Neo4j Ops Manager 1.15.1, included with Enterprise, supports any-to-any Neo4j
upgrades.

The server includes local-network deployment discovery (2026.05.0).
`neo4j-admin fleet discover` lists discovered servers, and `neo4j-admin` can
bulk-register them with Fleet Manager for Aura Console display.

## Cypher Shell

Cypher Shell defaults `--error-format` to `gql` (2025.06). Scripts that parse
stderr should set the flag explicitly when they require a different format.

From 2025.08, disable history for a session with:

```text
cypher-shell --history disable
```

The `:sysinfo` command supports Infinigraph deployments (2026.05.0).

## Server selection and packaged language defaults

From 2025.12, Enterprise settings `initial.server.allowed_databases` and
`initial.server.denied_databases` accept wildcard database-name patterns, and
their minimum value length drops from three characters to one.

Starting in 2026.02, packaged `neo4j.conf` explicitly sets
`db.query.default_language=CYPHER_25`. New deployments using that file default
newly created databases to Cypher 25. Retained and customized configurations
can differ.

From 2026.02.3, Community Edition includes a `web/` directory with Neo4j
Browser as a ZIP. Enterprise Edition has no `web/` directory and continues to
ship Browser as a JAR in `lib/`.

## Platform and package lifecycle

Neo4j 2025.01 removes support for macOS 11 and 12, Amazon Linux 2022 AMI,
Ubuntu Server 16.04, 18.04, and 20.04, and Windows Server 2016 and 2019.

RHEL 8.x, Debian 11.x, macOS 13 Ventura, and macOS 14 Sonoma are deprecated
from 2025.10 and supported only through the 2026 LTS. CentOS Stream 8.x and
SysV init scripts are deprecated from 2026.01.

From 2026.05, `debian:bullseye-slim` and
`redhat/ubi9-minimal:latest` are unsupported as base images. Ubuntu Server
22.04, macOS 15 Sequoia, CentOS Stream 9, and Windows Server 2022 are also
deprecated supported platforms (2026.05.0) and will be removed in a future
release.

## Routing and query-cache deprecations

The `server_policies` load-balancing plugin and
`dbms.routing.load_balancing.plugin` are deprecated from 2025.05.
`server.db.query_cache_size` is also deprecated. Do not introduce these
configuration entry points in new deployments.
