# CQL, Schema, Clients, and Tools

## Native protocol and value encoding

### Multiframe message limits

The configured CQL message-size limit applies to multiframe messages as well
as single-frame messages (since 5.0.3). Splitting a large logical message
across frames does not bypass the limit; clients must remain within it.

### Full UTF-8

`CBUtil` serializes the complete valid UTF-8 range correctly (since 5.0.3).
Clients can send valid UTF-8 values without compensating for the former
high-range serialization problem.

## Schema and type behavior

### Descending UDT and vector keys

Frozen UDTs and vectors can be descending clustering keys (since 5.0.3):

```cql
CREATE TYPE coordinates (x int, y int);
CREATE TABLE samples (
    sensor_id uuid,
    position frozen<coordinates>,
    embedding vector<float, 3>,
    PRIMARY KEY (sensor_id, position, embedding)
) WITH CLUSTERING ORDER BY (position DESC, embedding DESC);
```

Schema emitted for snapshots also contains definitions for UDTs used in these
reverse clustering positions.

### Materialized views in schema descriptions

`DESCRIBE TABLE` includes materialized views associated with the table (since
5.0.4). Schema-export tooling that consumes this output should expect the view
definitions rather than treating them as a separate missing artifact.

### Table-name validation

Cassandra rejects a table name when the corresponding generated filename
would be too long (since 5.0.6). Schema generators should shorten the
identifier after a validation failure rather than waiting for a filesystem
operation to fail.

### `BytesType` compatibility

`BytesType` is compatible only with scalar types (since 5.0.7). Schema
evolution and schema-management tools must not treat a non-scalar type as
compatible with `BytesType`.

## `CQLSSTableWriter`

### Production notifications

`CQLSSTableWriter` can notify its client whenever it produces an SSTable
(since 5.0.3). Callers can react at file-production time instead of discovering
new files only after the writer completes.

### Output format

The writer can select BTI or Big-format SSTables (since 5.0.5). Choose the
format through the writer rather than assuming all generated files use one
fixed format.

### Date and time vectors

Vectors whose elements are `date` or `time` serialize correctly through
`CQLSSTableWriter` (since 5.0.7). These vector element types no longer need a
client-side avoidance path.

## Logging and shell behavior

### Full Query Logging batches

Full Query Logging batch statements support null column-value tombstones
(since 5.0.4). A null representing a tombstone in a batch is a supported log
value rather than an invalid serialization case.

### Optional `cqlsh` history

`cqlsh` has an option to disable command history (since 5.0.7). Use it for
sessions that must not persist entered statements; do not assume history is
unconditionally written.

## Command-line tool environment

### Selective environment loading

`nodetool` and other tools do not source `cassandra-env.sh` when it is
unnecessary (since 5.0.5). Wrapper scripts must set any environment they need
explicitly rather than relying on unrelated side effects of that file.

### Direct I/O initialization

Tools skip the DirectIO check during initialization (since 5.0.4). A tool can
initialize without satisfying the server's Direct I/O check.

### Token metadata command

`nodetool checktokenmetadata` compares `TokenMetadata` with gossip endpoint
state (since 5.0.3):

```shell
nodetool checktokenmetadata
```

### Guardrail commands

Guardrail configuration is exposed through the finalized
`getguardrailsconfig` and `setguardrailsconfig` commands (since 5.0.5):

```shell
nodetool getguardrailsconfig
```

Use `setguardrailsconfig` with the guardrail setting to change.

### Corrected memory statistics

`nodetool gcstats` reports direct-memory usage correctly (since 5.0.7).
Consumers should interpret the corrected result rather than preserving a
compensation for the former value.

## Stress client TLS

`cassandra-stress` performs automatic TLS-version negotiation and supports TLS
1.3 by default (since 5.0.8). A TLS 1.3 endpoint does not require the client to
remain pinned to an older protocol version.

## Source builds and runtime

### Source distributions

Source distributions build with the Ant `artifacts` target, and the
native-protocol processing script used by that build is executable (since
5.0.5):

```shell
ant artifacts
```

### Java runtime

Cassandra has full Java 17 runtime support (since 5.0.5). Java 17 can be used
as the supported server runtime rather than only as a partial or experimental
build environment.
