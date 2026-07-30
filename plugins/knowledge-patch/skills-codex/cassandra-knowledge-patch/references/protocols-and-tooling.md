# Protocols and tooling

## Native protocol correctness

The configured CQL message-size limit applies to multiframe messages as well
as single-frame messages (5.0.3). A client cannot bypass the limit by splitting
one message across frames; clients should handle the resulting rejection.

`CBUtil` serializes the full valid UTF-8 range correctly (5.0.3). Remove
client-side exclusions that only worked around incomplete UTF-8 handling in
that utility.

## CQLSSTableWriter integration

`CQLSSTableWriter` can notify a client whenever it produces an SSTable
(5.0.3). Use the production callback when downstream processing must react to
files as they are emitted.

The writer can select BTI or Big-format SSTables (5.0.5). Make the format an
explicit integration choice when generated files must match a target
deployment.

Vectors whose element type is `date` or `time` are serialized correctly by the
writer (5.0.7).

## Logging and interactive clients

Full Query Logging batch statements accept null column values as tombstones
(5.0.4). Replay and inspection tooling must preserve the null rather than
treating it as an unsupported batch value.

`cqlsh` can disable command history (5.0.7). Use the option for sensitive or
ephemeral sessions where entered statements must not be persisted.

## Stress testing and TLS

`cassandra-stress` negotiates TLS versions automatically and supports TLS 1.3
by default (5.0.8). Avoid pinning an older TLS version merely to work around
the former client default.

## Source distribution builds

Source distributions can be built through the Ant `artifacts` target (5.0.5),
and the native-protocol processing script invoked by that build is executable:

```shell
ant artifacts
```
