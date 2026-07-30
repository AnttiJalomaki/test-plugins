# Upgrades and Breaking Changes

## Pre-upgrade checklist

Before upgrading:

1. Avoid the base 2025.06 release; its checkpoint mutex can sporadically
   deadlock. Use `2025.06.1` or later.
2. Complete the discovery v1-to-v2 transition before moving to Neo4j 2025.01.
3. Inventory removed settings, public Java APIs, administrative procedures,
   metrics, platform dependencies, and store formats.
4. Decide whether to retain the current configuration or adopt new-install
   defaults.
5. Test log, TLS, plan-output, and administrative-result parsers against the
   target release.

## Discovery v1 removal

Neo4j 2025.01 removes discovery service v1. Complete migration before the
upgrade. Internal discovery traffic moves from port `5000` to `6000`, and
settings move as follows:

```text
dbms.cluster.discovery.v2.endpoints -> dbms.cluster.endpoints
dbms.kubernetes.discovery.v2.service_port_name -> dbms.kubernetes.discovery.service_port_name
server.discovery.advertised_address -> server.cluster.advertised_address
server.discovery.listen_address -> server.cluster.listen_address
```

The old `*.v2.*` names remain accepted only for the 5.26-to-2025.01 migration
and should be replaced.

The discovery migration procedures
`dbms.cluster.moveToNextDiscoveryVersion()`,
`dbms.cluster.showParallelDiscoveryState()`, and
`dbms.cluster.switchDiscoveryServiceVersion()` are removed without
replacements. `dbms.setDatabaseAllocator()` is also removed without a
replacement.

## Server groups become tags

Catch-up strategies `connect-randomly-to-server-group` and
`connect-randomly-within-server-group` are replaced by their
`*-server-tags` forms. Rename configuration:

```text
db.cluster.raft.leader_transfer.priority_group -> db.cluster.raft.leader_transfer.priority_tag
server.cluster.catchup.connect_randomly_to_server_group -> server.cluster.catchup.connect_randomly_to_server_tags
server.groups -> initial.server.tags
```

## Replaced and removed settings

Neo4j 2025.01 has these replacements:

```text
db.logs.query.annotation_data_as_json_enabled -> db.logs.query.annotation_data_format
dbms.cluster.catchup.client_inactivity_timeout -> dbms.cluster.network.client_inactivity_timeout
server.max_databases -> dbms.max_databases
```

The following settings are removed without replacements:

```text
db.tx_state.memory_allocation
dbms.cluster.discovery.log_level
dbms.cluster.discovery.type
dbms.cluster.discovery.endpoints
dbms.cluster.discovery.version
dbms.kubernetes.service_port_name
initial.dbms.database_allocator
server.memory.off_heap.block_cache_size
server.memory.off_heap.max_cacheable_block_size
server.memory.off_heap.transaction_max_size
```

## New-install configuration defaults

These defaults apply to new installations and upgrades that replace existing
configuration files:

```text
db.logs.query.annotation_data_format: CYPHER -> JSON
server.metrics.csv.rotation.compression: NONE -> ZIP
server.panic.shutdown_on_panic: false -> true
server.logs.config: conf/server-logs.xml -> server-logs.xml
server.logs.user.config: conf/user-logs.xml -> user-logs.xml
```

Relative `server.logs.config` and `server.logs.user.config` paths resolve from
`server.directories.configuration`, not `server.directories.neo4j_home`.
The default `debug.log` format also changes from text to JSON. Keep its default
appender for supportability and add another appender for a second format.

## Removed public Java APIs

Neo4j 2025.01 removes public Java symbols tied to retired allocator, server
group, discovery, Raft, transaction-memory, and query-annotation facilities:

```text
EnterpriseEditionSettings.{initial_database_allocator,server_groups,server_max_number_of_databases}
WaitResponseState
ClusterSettings.{DEFAULT_CLUSTER_STATE_DIRECTORY_NAME,DEFAULT_DISCOVERY_PORT,DEFAULT_RAFT_PORT,DEFAULT_TRANSACTION_PORT,catchup_connect_randomly_to_server_group,raft_leader_transfer_priority_group}
ClusterBaseSettings.DEFAULT_DISCOVERY_PORT
ClusterNetworkSettings.catchup_client_inactivity_timeout
ParallelDiscoveryMode
RemotesResolver.Type and RemotesResolver.init(Type,Configuration,LogProvider)
ClusterAddressSettings.discovery_advertised_address
DiscoverySettings.{discovery_endpoints,discovery_listen_address,discovery_log_level,discovery_type,discovery_version}
KubernetesSettings.kubernetes_service_port_name
RaftSettings.{DEFAULT_CLUSTER_STATE_DIRECTORY_NAME,DEFAULT_RAFT_PORT}
SeedDownloadStreamWrapper and SeedProviderDependencies
GraphDatabaseSettings.{TransactionStateMemoryAllocation,log_queries_annotation_data_as_json,tx_state_max_off_heap_memory,tx_state_memory_allocation,tx_state_off_heap_block_cache_size,tx_state_off_heap_max_cacheable_block_size}
```

Replace removed `com.neo4j.dbms.seeding.SeedProvider` with
`DatabaseSeedProvider`.

## Procedure and command migrations

Replace cluster entry points:

```text
dbms.cluster.recreateDatabase() -> dbms.recreateDatabase()
dbms.cluster.routing.getRoutingTable() -> dbms.routing.getRoutingTable()
dbms.cluster.uncordonServer() -> ENABLE SERVER
dbms.cluster.readReplicaToggle() -> dbms.cluster.secondaryReplicationDisable()
dbms.quarantineDatabase() -> dbms.unquarantineDatabase()
```

The last two old procedures are removed in Cypher 25. Cypher 25 also removes
deprecated `dbms.upgrade()` and `dbms.upgradeStatus()`; remove those calls from
administrative automation.

Replace deprecated `database aggregate-backup` with
`neo4j-admin backup aggregate`. Replace `neo4j-admin database migrate
--page-cache` with `--max-off-heap-memory`.

## Metrics changes

Old `causal_clustering.core` Raft metrics covering indexes, term, leadership,
retries, in-flight cache, prefetch buffering, message processing, replication,
and last-leader messages are removed in favor of Raft metrics. The
`causal_clustering.read_replica.pull_update*` metrics move to store-copy
metrics, and discovery-v1 metrics under `cluster.discovery` are removed.

Rename `<prefix>.store.size.total` to `<prefix>.store.size.full`.

## Platform removals and deprecations

Neo4j 2025.01 removes macOS 11 and 12, Amazon Linux 2022 AMI, Ubuntu Server
16.04, 18.04, and 20.04, and Windows Server 2016 and 2019.

Later deprecations affect RHEL 8.x, Debian 11.x, macOS 13 and 14, CentOS Stream
8.x, SysV init, Ubuntu Server 22.04, macOS 15, CentOS Stream 9, and Windows
Server 2022. `debian:bullseye-slim` and `redhat/ubi9-minimal:latest` become
unsupported base images from 2026.05. Replace them before their support
windows end.

## TLS and seed compatibility

`dbms.ssl.policy.*.verify_hostname` defaults to `true` instead of `false`.
Verify certificates and peer names before accepting the new default.

`URLConnectionSeedProvider` no longer supports `file` locations in either
Cypher 5 or Cypher 25. Use `FileSeedProvider`. Replace `S3SeedProvider` with
`CloudSeedProvider` from 5.26.

## Store-format deadline

The next LTS is the final release able to read, write, or migrate `high_limit`
databases (2026.06.0). Migrate them offline to Block format before moving
beyond it. A remaining `high_limit` database will fail to start without a
compatibility fallback.

The `standard` format has been deprecated since 5.23. Avoid it for new
databases and plan migration for existing stores.
