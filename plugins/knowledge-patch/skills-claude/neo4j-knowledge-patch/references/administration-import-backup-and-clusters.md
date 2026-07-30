# Administration, Import, Backup, and Clusters

Use this reference for `neo4j-admin`, Cypher administration commands, database
seeding, cluster allocation, Fleet Manager, Ops Manager, and Cypher Shell.

## Database seeding

### Current option contract

Cypher 25 changes database-seed options as follows:

- `seedCredentials` is removed. Supply cloud-seed credentials through the
  cloud provider's built-in credential mechanism.
- `existingDataSeedInstance` becomes `existingDataSeedServer`.
- `seedSourceDatabase` filters the restored backup artifacts.
- `existingData` is deprecated and is now optional.
- `CREATE DATABASE` accepts Java `Long` parameters as well as `Int` parameters.

`S3SeedProvider` is replaced by `CloudSeedProvider` from 5.26. For filesystem
seeds, use `FileSeedProvider`: `URLConnectionSeedProvider` no longer supports
`file` locations in either Cypher 5 or Cypher 25.

### Cluster-local sharded database seeds

A sharded property database can be seeded from artifacts in cluster members'
seed-repository folders. Supply `server://` locations in `seedUri` (since
2026.04.0):

```cypher
CREATE DATABASE spd OPTIONS {
  seedUri: ["server://server-1/", "server://server-2/"]
}
```

## Import

### Backup-format and identity options

`neo4j-admin database import` and `neo4j-admin database copy` accept
`--compress` when `--target-format=backup` produces backup-format output.

Multi-column identities can use `INTEGER` ID types instead of being forced to
`STRING`, so composite keys retain their intended type.

### Graph-type schema during import

For a full import, the preview command below can be passed through `--schema`:

```cypher
ALTER CURRENT GRAPH TYPE SET { ... }
```

For an incremental import, `--schema` accepts graph-type additions, removals,
and alterations (since 2026.06.0):

```cypher
ALTER CURRENT GRAPH TYPE ADD { ... }
ALTER CURRENT GRAPH TYPE DROP { ... }
ALTER CURRENT GRAPH TYPE ALTER { ... }
```

This extends graph-type schema changes beyond the full-import path.

### Bad-entry tolerance

From 2025.12, both `neo4j-admin database import full` and `incremental` default
`--bad-tolerance` to `-1`, meaning unlimited, instead of `1000`. Specify a
finite value when import should stop after a bounded number of malformed
entries.

### Progress log locations

Import progress changes location across releases:

- In 2026.03 it moves from
  `server/logs/neo4j-admin-import-yyyy-MM-dd.HH.mm.ss.log` to
  `server/data/imports/dbname-yyyy-MM-dd.HH.mm.ss/import.log`.
- In 2026.04 the generated import-information directory moves back under
  `server/logs/`.

Determine the location from the installed release rather than hard-coding a
single path.

Vector-specific import parsing is documented in the vector reference.

## Database copy and migration memory

From 2025.01, `neo4j-admin database copy --from-pagecache=<size>` limits
off-heap memory for the entire copy operation, covering reads and writes, not
only the source read cache. The clearer equivalent is:

```text
--max-off-heap-memory=<size>
```

For `neo4j-admin database migrate`, replace the deprecated `--page-cache`
option with the same `--max-off-heap-memory` option.

## Backup operations

### Inspection order

`neo4j-admin backup inspect` orders output by append index. If entries share an
append index, time breaks the tie. Consumers that depend on inspection order
should use that contract.

### Aggregate command

Replace the deprecated:

```text
neo4j-admin database aggregate-backup
```

with:

```text
neo4j-admin backup aggregate
```

### User-filtered metadata

From 2025.10, include only named users and their role assignments in backup
metadata with:

```text
--include-metadata=users=alice,bob
```

## Page-cache I/O

Linux deployments can opt into initial `io_uring` support for the background
page evictor and checkpointer (since 2026.04.0):

```properties
server.memory.pagecache.async=true
```

Validate kernel, filesystem, and workload behavior before enabling the option
in production.

## Administrative result contracts

Cypher 25 changes several results and errors:

- `SHOW TRANSACTIONS.startTime` and `currentQueryStartTime` are
  `ZONED DATETIME`, not `STRING`.
- Unavailable values in several transaction columns are `null`.
- Administration commands with `WAIT` report cluster state through
  notifications instead of result rows.
- Revoking a privilege that cannot exist raises an error.

Update deserializers, scripts, and tests accordingly.

## Allocation and topology

From 2025.12 in Cypher 25, `dbms.setDefaultAllocationNumbers()` accepts the
additional `propertyShardReplicas` input, and
`dbms.showTopologyGraphConfig()` returns `propertyShardReplicas`.

Enterprise Edition settings `initial.server.allowed_databases` and
`initial.server.denied_databases` accept wildcard database-name patterns from
2025.12. Their minimum value length is reduced from three characters to one.

## Server-management and cluster procedures

`dbms.cluster.cordonServer()`,
`dbms.cluster.setAutomaticallyEnableFreeServers()`, and
`dbms.cluster.uncordonServer()` require `SERVER MANAGEMENT`. Relying on a broad
admin privilege for these calls is deprecated; grant the specific privilege.

Migrate cluster procedure calls as follows:

```text
dbms.cluster.recreateDatabase() -> dbms.recreateDatabase()
dbms.cluster.routing.getRoutingTable() -> dbms.routing.getRoutingTable()
dbms.cluster.uncordonServer() -> ENABLE SERVER
```

Cypher 25 also replaces or removes:

```text
dbms.cluster.readReplicaToggle() -> dbms.cluster.secondaryReplicationDisable()
dbms.quarantineDatabase() -> dbms.unquarantineDatabase()
```

`dbms.setDatabaseAllocator()` is removed without replacement. The deprecated
`dbms.upgrade()` and `dbms.upgradeStatus()` procedures are also removed in
Cypher 25; automation must not invoke them.

## Fleet and operations management

### Discover and register local deployments

The server includes a local-network discovery service. Run:

```text
neo4j-admin fleet discover
```

to list discovered servers. `neo4j-admin` can then bulk-register them with
Fleet Manager for display in the Aura Console.

Enterprise Fleet Management is no longer bundled separately with the DBMS
package because fleet-management capability is included in Neo4j.

Neo4j Ops Manager 1.15.1, included with Enterprise, supports any-to-any Neo4j
upgrades.

## Cypher Shell

From 2025.08, disable shell history for a session with:

```text
cypher-shell --history disable
```

Cypher Shell now defaults `--error-format` to `gql`. Scripts that require a
different error representation should pass the option explicitly.

The `:sysinfo` command supports Infinigraph deployments.
