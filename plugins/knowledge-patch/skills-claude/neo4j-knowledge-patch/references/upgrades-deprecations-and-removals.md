# Upgrades, Deprecations, and Removals

Use this reference to prepare an upgrade and to identify migration work that
must be completed before changing binaries.

## Release and store-format blockers

- The base 2025.06 release can sporadically deadlock on the checkpoint mutex.
  Run 2025.06.1 or later in production.
- The next LTS is the final release able to read, write, or migrate a
  `high_limit` database. Before upgrading beyond that LTS, migrate the database
  offline to Block format. Beyond the deadline, a remaining `high_limit`
  database fails to start and has no compatibility fallback.
- The `standard` store format has been deprecated since 5.23. Do not select it
  for new databases, and plan to migrate existing stores.

## Discovery-v2 prerequisite

Neo4j 2025.01 removes discovery service v1. A cluster must finish its v1-to-v2
transition before the upgrade. Internal discovery traffic moves from port
`5000` to `6000`.

Replace these settings:

```text
dbms.cluster.discovery.v2.endpoints -> dbms.cluster.endpoints
dbms.kubernetes.discovery.v2.service_port_name -> dbms.kubernetes.discovery.service_port_name
server.discovery.advertised_address -> server.cluster.advertised_address
server.discovery.listen_address -> server.cluster.listen_address
```

The old `*.v2.*` forms remain accepted only to support migration from 5.26 to
2025.01. Replace them after the transition. The following discovery migration
procedures are removed without replacements:

```text
dbms.cluster.moveToNextDiscoveryVersion()
dbms.cluster.showParallelDiscoveryState()
dbms.cluster.switchDiscoveryServiceVersion()
```

## Server groups become tags

Replace catch-up strategies `connect-randomly-to-server-group` and
`connect-randomly-within-server-group` with the corresponding
`*-server-tags` strategies. Move configuration as follows:

```text
db.cluster.raft.leader_transfer.priority_group -> db.cluster.raft.leader_transfer.priority_tag
server.cluster.catchup.connect_randomly_to_server_group -> server.cluster.catchup.connect_randomly_to_server_tags
server.groups -> initial.server.tags
```

## Configuration replacements and removals

Neo4j 2025.01 makes these direct replacements:

```text
db.logs.query.annotation_data_as_json_enabled -> db.logs.query.annotation_data_format
dbms.cluster.catchup.client_inactivity_timeout -> dbms.cluster.network.client_inactivity_timeout
server.max_databases -> dbms.max_databases
```

Remove the following settings; they have no replacement:

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

Also migrate away from these deprecated entry points:

- The `server_policies` load-balancing plugin and
  `dbms.routing.load_balancing.plugin` are deprecated from 2025.05.
- `server.db.query_cache_size`,
  `dbms.security.oidc.<provider>.auth_params`, and
  `dbms.security.oidc.<provider>.client_id` are deprecated.

## Defaults that change with configuration replacement

These defaults apply to new installations and to upgrades that replace the
existing configuration files:

```text
db.logs.query.annotation_data_format: CYPHER -> JSON
server.metrics.csv.rotation.compression: NONE -> ZIP
server.panic.shutdown_on_panic: false -> true
server.logs.config: conf/server-logs.xml -> server-logs.xml
server.logs.user.config: conf/user-logs.xml -> user-logs.xml
```

Relative `server.logs.config` and `server.logs.user.config` values are now
resolved from `server.directories.configuration`, not
`server.directories.neo4j_home`.

The default `debug.log` format also changes from text to JSON. Keep the default
appender for supportability and add a second appender if another format is
required. Update consumers that parse the default file.

## Removed public Java APIs

Neo4j 2025.01 removes public Java symbols associated with retired allocator,
server-group, discovery, Raft, transaction-memory, seeding, and query-annotation
facilities. Recompile extensions and replace imports for:

```text
EnterpriseEditionSettings.{initial_database_allocator,server_groups,server_max_number_of_databases}
WaitResponseState
ClusterSettings.{DEFAULT_CLUSTER_STATE_DIRECTORY_NAME,DEFAULT_DISCOVERY_PORT,DEFAULT_RAFT_PORT,DEFAULT_TRANSACTION_PORT,catchup_connect_randomly_to_server_group,raft_leader_transfer_priority_group}
ClusterBaseSettings.DEFAULT_DISCOVERY_PORT
ClusterNetworkSettings.catchup_client_inactivity_timeout
ParallelDiscoveryMode
RemotesResolver.Type
RemotesResolver.init(Type,Configuration,LogProvider)
ClusterAddressSettings.discovery_advertised_address
DiscoverySettings.{discovery_endpoints,discovery_listen_address,discovery_log_level,discovery_type,discovery_version}
KubernetesSettings.kubernetes_service_port_name
RaftSettings.{DEFAULT_CLUSTER_STATE_DIRECTORY_NAME,DEFAULT_RAFT_PORT}
SeedDownloadStreamWrapper
SeedProviderDependencies
GraphDatabaseSettings.{TransactionStateMemoryAllocation,log_queries_annotation_data_as_json,tx_state_max_off_heap_memory,tx_state_memory_allocation,tx_state_off_heap_block_cache_size,tx_state_off_heap_max_cacheable_block_size}
```

The removed `com.neo4j.dbms.seeding.SeedProvider` has the explicit replacement
`DatabaseSeedProvider`.

## Metrics migrations

- The old `causal_clustering.core` Raft metric family is removed. This includes
  its index, term, leadership, retry, in-flight-cache, prefetch-buffer,
  message-processing, replication, and last-leader series. Use the current Raft
  metrics.
- The three `causal_clustering.read_replica.pull_update*` metrics move to the
  store-copy metrics namespace.
- The six discovery-v1 metrics under `cluster.discovery` are removed without
  replacements.
- `<prefix>.store.size.total` is renamed to `<prefix>.store.size.full`; update
  dashboards and alerts.

See the observability reference for separately deprecated Raft status fields
and metrics.

## Removed platform support

Neo4j 2025.01 no longer supports:

- macOS 11 or 12;
- the Amazon Linux 2022 AMI;
- Ubuntu Server 16.04, 18.04, or 20.04; or
- Windows Server 2016 or 2019.

Additional platforms are on a retirement path:

- RHEL 8.x, Debian 11.x, macOS 13 Ventura, and macOS 14 Sonoma are deprecated
  from 2025.10 and supported only through the 2026 LTS.
- CentOS Stream 8.x and SysV init scripts are deprecated from 2026.01.
- `debian:bullseye-slim` and `redhat/ubi9-minimal:latest` are unsupported as
  base images from 2026.05.
- Ubuntu Server 22.04, macOS 15 Sequoia, CentOS Stream 9, and Windows Server
  2022 are deprecated as supported platforms and will be removed in a future
  release.

## Upgrade audit

1. Patch 2025.06 before production use.
2. Complete discovery migration and change the cluster port and setting names.
3. Replace server groups, configuration keys, removed procedures, metrics, and
   Java symbols.
4. Migrate legacy store formats before their final readable release.
5. Decide whether to preserve existing configuration files or adopt the new
   defaults, paths, and JSON logging behavior.
6. Validate the operating system, base image, init system, TLS policy, plugins,
   and client integrations against their current support paths.
