# Networking and accounts

## Leafnode TLS-first handshakes

Since 2.11.0, a leafnode TLS block can negotiate TLS before any NATS protocol
handshake:

```text
tls {
  handshake_first: true
}
```

## Route and gateway reconnect backoff

Since 2.12.0, routes and gateways can set `connect_backoff: true`. Reconnect
delays then increase exponentially from one to 30 seconds. This reduces DNS
and connection storms during restart or outage events, with slower
reconnection as the tradeoff.

## Leafnode interest isolation and reload

Since 2.12.0, `isolate_leafnode_interest` stops east-west interest propagation
between leafnodes when they do not need to communicate directly.

A solicited leafnode remote can be disconnected and suppressed on reload with
`disabled: true`, then re-enabled by reloading it as false. Since 2.14.0, the
entire leafnode remotes section can also be added or removed on configuration
reload without restarting the server.

## Per-account replication traffic

Since 2.11.0, the per-account JetStream setting `cluster_traffic` can move an
asset's Raft replication traffic from the system account to the account that
owns the asset. With multiple route connections, this can reduce cross-account
head-of-line blocking.

## Global-account system events

Since 2.12.0, the global `$G` account produces system events, including client
connect and disconnect events.

## Server and connection metadata

Since 2.12.0, `server_metadata` adds arbitrary string key/value metadata
alongside `server_tags`.

Server statistics also report effective `GOMAXPROCS` and `GOMEMLIMIT`.
Client-related log records include account and user names, and
server-connection close logs include the remote server name.

## TLS cipher-suite defaults

Since 2.12.0, cipher suites newly available through Go's `crypto/tls` are
picked up automatically, while insecure suites are disabled by default. Set
`allow_insecure_cipher_suites` only when a legacy peer still requires them.

## MQTT Sparkplug B awareness

Since 2.11.0, the built-in MQTT service is Sparkplug B Aware and handles
`NBIRTH` and `NDEATH` messages.

## Name validation and process status

Since 2.11.0, spaces in server, cluster, or gateway names cause startup to
fail. Graceful shutdown by `SIGTERM` exits with status `0`; supervisors should
not classify that status as an error.
