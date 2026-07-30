---
name: cassandra-knowledge-patch
description: Apache Cassandra
version: 5.0.8
license: MIT
metadata:
  author: Nevaberry
---


# Apache Cassandra Knowledge Patch

Use this skill when changing, upgrading, operating, or integrating with Apache
Cassandra and the work touches the behaviors documented here. Start with the
quick checks below, then open the topic reference that matches the task.

## Reference index

| Reference | Topics |
| --- | --- |
| [security-and-configuration.md](references/security-and-configuration.md) | Authorizers, permissions, audit logging, virtual settings, guardrails, and configuration compatibility |
| [queries-schema-and-indexing.md](references/queries-schema-and-indexing.md) | CQL semantics, schema validation, SAI, legacy indexes, filtering, and reconciliation |
| [storage-and-data-path.md](references/storage-and-data-path.md) | SSTables, commitlog, hints, mutation serialization, compaction, snapshots, and streaming |
| [operations-topology-and-repair.md](references/operations-topology-and-repair.md) | Gossip, JMX, node tools, runtime support, failure detection, repair, and AutoRepair |
| [protocols-and-tooling.md](references/protocols-and-tooling.md) | Native protocol limits, client serialization, CQLSSTableWriter, FQL, cqlsh, stress, and builds |

## Upgrade triage

Check these compatibility boundaries before treating a changed result as a
regression:

1. Re-test authorization. Access formerly admitted through DC, authorizer, or
   system-keyspace edge cases can now be rejected.
2. Re-test identity provisioning. A non-superuser cannot bind an identity to a
   superuser, and rapid password changes can be rate-limited.
3. Parse complex values from `system_views.settings` as JSON, and never depend
   on that view to disclose security-sensitive values.
4. Validate generated table names before issuing DDL; Cassandra now rejects
   names that would create overlong filenames.
5. Re-test schema-compatibility tooling that treats `BytesType` as broadly
   compatible. Compatibility is limited to scalar types.
6. Do not rely on `nodetool` or other tools sourcing `cassandra-env.sh` as an
   incidental side effect.
7. During a mixed-major-version upgrade, do not assume the in-process repair
   scheduler remains active; it stops when it detects two major versions.
8. If failure-detection timing changes with an otherwise default
   configuration, account for the corrected maximum-interval calculation.

## Authorization and configuration quick checks

### Authorizers and system keyspaces

- Preserve parameters under `CassandraCIDRAuthorizer`; configured parameters
  are now applied.
- Expect malformed `audit_logging_options` to fail validation during startup.
- Monitoring roles may receive `SELECT` on `system_views` and
  `system_virtual_schema`.
- Treat password-change throttling as an expected rejection path and back off
  in provisioning automation.

### Settings consumers

- Deserialize complex `system_views.settings` values as JSON.
- Treat sensitive values as intentionally redacted.
- Do not assume an absent configuration is unsupported; settings omitted by
  an earlier view implementation are exposed again.
- Optional entries in the default `cassandra.yaml` should remain parseable
  when uncommented; preserve valid YAML in downstream templates.

### Guardrails

- Inspect or update guardrail settings with `nodetool getguardrailsconfig` and
  `nodetool setguardrailsconfig`.
- A disk-usage guardrail can be disabled after it has tripped.
- A configured disk-usage maximum no longer makes first boot depend on the
  data directory already existing.

## Query and index correctness quick checks

### SAI lifecycle

- A restarted node does not become fully query-ready until its built SAI
  indexes are queryable; the startup ordering now avoids an availability gap.
- Repair flushes of partial partitions or rows mark SAI as non-empty.
- An SSTable-writer switch flushes the active SAI segment builder.
- ANN execution uses score-ordered iterators; verify result ordering and
  latency expectations together.
- Use `nodetool tablestats` when selected SAI state and query-performance
  metrics are needed.

### Mixed indexes and filters

- If legacy 2i and SAI both index a column, legacy 2i is selected first.
- Empty values remain invalid for non-literal and other types that disallow
  them; do not depend on an index accepting such values.
- Re-test intersection queries involving repaired matches, multiple
  non-indexed predicates, static columns, or composite map values.
- Preserve deletions when result reconciliation is required; filtering no
  longer purges them early.

### Schema and expressions

- UDT and vector clustering columns support `DESC` ordering.
- `min` and `max` over descending clustering columns follow the corrected
  semantics.
- `DESCRIBE TABLE` includes the table's materialized views.
- Expect a validation error for table identifiers that would overflow the
  filesystem filename limit.

## Storage and data-path quick checks

### Hints and mutations

- Hint expiry is based on request start time, not timeout time.
- Schema mismatch alone no longer blocks hint delivery.
- Multi-collection row serialization preserves complex deletions.
- Updating a column with a new TTL but the same expiration time is
  deterministic across replicas.

### SSTables and commitlog

- Direct I/O commitlog flushes use the corrected safe path.
- Memory-mapped trie indexes larger than 2 GiB are readable.
- Early-open BTI SSTables return correct range-query results.
- A corrupt SSTable encountered during compaction is marked suspected and its
  buffer-pool resources are released.
- Commitlog recovery skips sync blocks correctly after CRC errors.
- Legacy SSTables with the old Bloom-filter format automatically fall back
  from zero-copy streaming.

### Snapshots and compaction

- Snapshot paths accept names that were previously rejected too broadly.
- Snapshot schema CQL carries definitions for UDTs used by reverse clustering
  columns.
- Unified Compaction rejects invalid minimum/target size combinations.
- Snapshot loading handles pre-table-ID directory layouts.

## Operations and topology quick checks

### Gossip and metadata

- Delayed shutdown state cannot overwrite fresh startup state after a restart.
- Gossip-only and bootstrapping nodes receive DC, rack, and host-ID state.
- Concurrent multi-field endpoint updates converge.
- Run `nodetool checktokenmetadata` to compare `TokenMetadata` with gossip
  endpoint state.

### Management interfaces

- `StorageService` is available over JMX during bootstrap.
- JMX can drop prepared statements through
  `StorageService.dropPreparedStatements`.
- `StorageProxyMBean` exposes the per-IP native transport connection cap.
- `nodetool gcstats` reports corrected direct-memory usage; adjust monitoring
  baselines rather than compensating for the older value.

### Repair

- Long-running repairs are not failed merely for running too long.
- The built-in AutoRepair scheduler supports recurring in-process repair,
  task-duration bounds, `preview_repaired`, disk protection for full repair,
  and expected-versus-actual progress reporting.
- Keep external repair orchestration aware of scheduler shutdown during
  mixed-major-version operation.

## Protocol and tool quick checks

- Enforce the CQL message-size limit across an entire multiframe message.
- Preserve the complete UTF-8 range when using Cassandra's binary utilities.
- `CQLSSTableWriter` can notify on produced files, choose BTI or Big format,
  and serialize vectors containing `date` or `time` elements.
- Full Query Logging batches preserve null value tombstones.
- Use the no-history option for `cqlsh` sessions that must not persist entered
  statements.
- `cassandra-stress` negotiates TLS 1.3 by default.
- Java 17 is supported, and source distributions can be produced with
  `ant artifacts`.

## Task workflow

1. Identify whether the task is primarily security/configuration, query/index,
   storage/data-path, operations/repair, or protocol/tooling work.
2. Read the corresponding reference in full; cross-read adjacent references
   when a repair, restart, or upgrade crosses subsystem boundaries.
3. Compare the project configuration and automation with the compatibility
   boundary described there.
4. Prefer current observed behavior and repository tests over assumptions
   encoded in older scripts or monitoring rules.
5. For operational changes, verify both correctness and observability: command
   output, JMX attributes, virtual tables, and repair progress may all have
   changed together.
