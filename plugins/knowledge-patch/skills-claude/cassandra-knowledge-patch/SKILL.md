---
name: cassandra-knowledge-patch
description: Apache Cassandra
version: 5.0.8
license: MIT
metadata:
  author: Nevaberry
---


# Apache Cassandra Knowledge Patch

Use this skill when designing, upgrading, operating, or debugging Apache
Cassandra 5.0 deployments. It focuses on behavior that is easy to miss in
configuration, repair, indexing, storage, CQL, JMX, and command-line tooling.

## How to use this skill

1. Identify the affected surface: configuration and authorization, topology
   and repair, indexes and reads, storage and recovery, or CQL and tools.
2. Read the matching reference file before changing configuration or writing
   compatibility logic.
3. Check mixed-version behavior explicitly during rolling upgrades.
4. Treat corrected rejection, validation, and redaction behavior as an API
   change for automation and monitoring consumers.
5. Prefer observed node state, JMX state, and command output over assumptions
   inherited from an older patch level.

## Reference index

| Reference | Topics |
| --- | --- |
| [configuration-security-observability.md](references/configuration-security-observability.md) | YAML validation, authorizers, permissions, guardrails, virtual settings, JMX, failure handling, metrics |
| [repair-topology-coordination.md](references/repair-topology-coordination.md) | AutoRepair, manual repair, gossip, token metadata, hints, Paxos, batchlog, streaming |
| [indexing-and-query-correctness.md](references/indexing-and-query-correctness.md) | SAI, legacy secondary indexes, ANN, row filtering, static rows, clustering order, aggregates |
| [storage-sstables-and-recovery.md](references/storage-sstables-and-recovery.md) | SSTable formats, snapshots, compaction, commitlog, Direct I/O, caches, deletion and TTL reconciliation |
| [cql-schema-clients-and-tools.md](references/cql-schema-clients-and-tools.md) | Native protocol, schema rules, data types, `CQLSSTableWriter`, FQL, `cqlsh`, build and tool behavior |

## Compatibility changes to check first

### Authorization is stricter

- Access around data centers, authorizers, and system keyspaces that was
  inadvertently accepted can now be rejected.
- A regular user cannot bind an identity to a superuser.
- Password changes are rate-limited.
- `BytesType` compatibility applies only to scalar types.

Audit provisioning, role-management, and schema-evolution automation for
assumptions that depended on permissive behavior.

### Configuration inventory is safer but different

`system_views.settings` represents complex settings as JSON and redacts
security-sensitive values. Configurations that had temporarily disappeared
from the view are present again. Inventory collectors should parse complex
values as JSON, tolerate redacted secrets, and avoid treating the virtual
table as a secret source.

### Tool startup has fewer environment side effects

`nodetool` and related tools do not always source `cassandra-env.sh`, and tool
initialization skips the DirectIO check. Do not depend on unrelated shell side
effects merely because a Cassandra tool was invoked.

### Validation can fail earlier

- Audit logging options are sanitized and validated at startup.
- Invalid Unified Compaction size combinations are rejected.
- Table names that would create overlong filenames are rejected during DDL.
- A configured disk-usage maximum no longer breaks first boot before the data
  directory exists.

### Operational artifacts and defaults changed

- Handled exceptions no longer produce heap dumps.
- The failure detector's default maximum interval is calculated correctly and
  may change failure-detection timing.
- `nodetool gcstats` reports corrected direct-memory values.
- Command history in `cqlsh` can be disabled.

## Automated repair quick reference

Built-in AutoRepair provides an in-process scheduler for recurring repairs.
When enabling it, account for all of these behaviors:

- A minimum repair-task duration can bound scheduled task runtime.
- `preview_repaired` is supported as a repair type.
- The scheduler stops when two major Cassandra versions are detected.
- Full AutoRepair observes disk-protection conditions.
- Progress reporting compares expected and actual repair bytes and keyspaces.

During a mixed-major-version upgrade, provide another repair plan because the
scheduler intentionally stops. For full repairs, confirm disk protection is
not preventing work before treating lack of progress as a scheduler failure.
See
[repair-topology-coordination.md](references/repair-topology-coordination.md).

## Index and query quick reference

### SAI lifecycle

Already-built SAI indexes become queryable without a post-restart gap. Repair
flushes mark affected indexes non-empty, and switching an SSTable writer
flushes the active SAI segment builder. These lifecycle fixes matter when
diagnosing transient empty results or index availability around restart,
repair, and writer boundaries.

### Query correctness

Corrected cases include:

- intersections spanning repaired index matches and multiple non-indexed
  columns;
- composite non-indexed columns containing maps;
- queries involving static columns;
- numeric range intersections on one column;
- approximate-nearest-neighbor execution using score-ordered iterators;
- range reads from early-open BTI SSTables.

When a legacy secondary index and SAI both cover a column, Cassandra chooses
the legacy secondary index first. Empty values remain invalid for types that
do not permit them.

### Descending clustering

UDTs and vectors can be descending clustering keys. `min` and `max` return
correct values for descending clustering columns, and snapshot-generated CQL
includes UDT definitions needed by reverse clustering columns.

See
[indexing-and-query-correctness.md](references/indexing-and-query-correctness.md)
for the complete query and index behavior.

## Operations quick reference

### Validate topology state

Use `nodetool checktokenmetadata` when `TokenMetadata` may have diverged from
gossip endpoint state:

```shell
nodetool checktokenmetadata
```

Gossip-only and bootstrapping nodes receive DC, rack, and host-ID state.
Restarted nodes are protected from delayed shutdown messages, and concurrent
multi-field endpoint-state updates converge correctly.

### Observe indexes and guardrails

`nodetool tablestats` exposes selected SAI state and query-performance data:

```shell
nodetool tablestats
```

Guardrail configuration is available through dedicated commands:

```shell
nodetool getguardrailsconfig
```

The disk-usage guardrail can be disabled even after its failure threshold has
tripped. See the configuration reference before wiring command output into
automation.

### Use management interfaces during bootstrap

The `StorageService` MBean is available while a node bootstraps.
`StorageService.dropPreparedStatements` is exposed through JMX, and
`StorageProxyMBean` exposes the per-IP native connection limit.

## Storage and recovery quick reference

- Direct I/O commitlog flushes are safe.
- Commitlog recovery skips sync blocks correctly after CRC errors.
- A corrupt SSTable encountered by compaction is marked suspect and its
  buffer-pool resources are released.
- Memory-mapped trie indexes larger than 2 GiB are readable.
- Zero-copy streaming automatically falls back for legacy SSTables with the
  old Bloom-filter format.
- Snapshot loading accepts pre-2.1 schema directories without table IDs.
- Snapshot-name path validation no longer rejects otherwise valid names.

Deletion and expiration semantics also received correctness fixes. Preserve
complex collection deletions, reconciliation-required deletions, expired-row
secondary-index notifications, and deterministic same-expiration TTL updates.
See
[storage-sstables-and-recovery.md](references/storage-sstables-and-recovery.md).

## Client and schema quick reference

- The CQL size limit applies to multiframe as well as single-frame messages.
- Full UTF-8 values serialize correctly.
- Full Query Logging batches accept null column-value tombstones.
- `DESCRIBE TABLE` includes materialized views.
- `CQLSSTableWriter` can report produced SSTables, select BTI or Big format,
  and serialize vectors containing `date` or `time`.
- `cassandra-stress` negotiates TLS automatically and supports TLS 1.3 by
  default.
- Source distributions build with `ant artifacts`.
- Running Cassandra with Java 17 is fully supported.

See [cql-schema-clients-and-tools.md](references/cql-schema-clients-and-tools.md)
for compatibility details.

## Upgrade and incident checklist

Before or during an upgrade:

1. Test role and identity provisioning against stricter authorization.
2. Confirm virtual-settings consumers handle JSON and redaction.
3. Do not depend on `cassandra-env.sh` being loaded by tools.
4. Plan for AutoRepair to stop during mixed-major-version operation.
5. Recheck failure-detector timing if using its default maximum interval.
6. Exercise mixed-version Paxos and legacy-SSTable streaming paths.

When investigating incorrect or missing data:

1. Separate SAI lifecycle problems from query-planning problems.
2. Check whether repair, early-open BTI files, static rows, or collection
   deletions match a corrected edge case.
3. Inspect token metadata and gossip convergence for topology symptoms.
4. Review commitlog CRC handling and suspected SSTables for recovery symptoms.
5. Verify that TTL and reconciliation behavior is deterministic across
   replicas.

When investigating stalled maintenance:

1. Determine whether AutoRepair intentionally stopped for mixed major
   versions.
2. Check disk protection before retrying a full automated repair.
3. Use expected-versus-actual AutoRepair progress fields.
4. Do not classify a long-running manual repair as failed merely because it
   exceeds an older implicit duration expectation.
