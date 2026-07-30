# Networking, Security, and Observability

## Distributed message tracing

Since 2.11.0, a publisher can request hop events as a message:

- enters or leaves a server;
- crosses a connection type;
- crosses an account boundary;
- undergoes subject mapping.

Set `Nats-Trace-Dest` to an inbox that will receive the events. Add
`Nats-Trace-Only: true` to propagate trace events without delivering the
traced message to its subscribers.

```text
Nats-Trace-Dest: trace.inbox
Nats-Trace-Only: true
```

Since 2.14.0, distributed message tracing preserves an existing `traceparent`
header instead of modifying it.

## JetStream replication traffic

Since 2.11.0, the per-account JetStream setting `cluster_traffic` can move an
asset's Raft replication traffic from the system account into the account that
owns the asset. In deployments with multiple route connections, this can
reduce cross-account head-of-line blocking.

## Routes and gateways

Since 2.12.0, routes and gateways accept:

```text
connect_backoff: true
```

This applies exponential reconnect delays from one to 30 seconds, reducing
DNS and connection storms during restarts or outages. The tradeoff is a slower
reconnection.

## Leafnodes

### TLS-first connections

Since 2.11.0, a leafnode TLS block can negotiate TLS before any NATS protocol
handshake:

```text
tls {
  handshake_first: true
}
```

### Interest isolation and remote reloads

Since 2.12.0, `isolate_leafnode_interest` prevents east-west interest
propagation between leafnodes that do not need to communicate directly.

A solicited remote can be disconnected and suppressed on configuration reload
with `disabled: true`, then re-enabled by reloading it as false. Since 2.14.0,
the entire leafnode remotes section can also be added or removed on reload
without restarting the server.

## TLS cipher suites

Since 2.12.0, cipher suites added by Go's `crypto/tls` are picked up
automatically, while insecure suites are disabled by default. Set
`allow_insecure_cipher_suites` only when a legacy peer still requires one of
those suites.

## Filestore encryption on Windows

Since 2.11.0, Windows deployments can protect JetStream filestore encryption
keys with the machine TPM. This avoids relying only on storage that remains
accessible to an attacker with physical access.

## MQTT Sparkplug B

Since 2.11.0, the built-in MQTT service is Sparkplug B Aware and handles
`NBIRTH` and `NDEATH` messages.

## Server and connection metadata

Since 2.12.0, `server_metadata` adds arbitrary string key/value metadata
alongside `server_tags`.

Server statistics also report effective `GOMAXPROCS` and `GOMEMLIMIT`.
Client-related logs include the account and user names, and server-connection
close logs include the remote server name.

## Global-account system events

Since 2.12.0, the global `$G` account produces system events, including client
connect and disconnect events.

## Startup name validation

Since 2.11.0, server, cluster, and gateway names containing spaces are
rejected at startup. Validate generated names before deploying a
configuration.
