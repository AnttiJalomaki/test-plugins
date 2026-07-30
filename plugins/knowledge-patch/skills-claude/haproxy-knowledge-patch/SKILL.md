---
name: haproxy-knowledge-patch
description: HAProxy
version: 3.4.0
license: MIT
metadata:
  author: Nevaberry
---


# HAProxy Knowledge Patch

Use this skill when configuring, upgrading, operating, or extending current
HAProxy deployments. Start with the breaking-change notes, then load the topic
reference that matches the task. Validate configuration with the exact binary
that will run it, especially when deprecated or experimental directives are
involved.

## Reference index

| Reference | Topics |
| --- | --- |
| [Upgrades and operations](references/upgrades-and-operations.md) | Breaking defaults, deprecations, reloads, CPU placement, builds, support branches |
| [TLS, QUIC, and networking](references/tls-quic-and-networking.md) | Certificates, ACME, SNI, QUIC/H3/QMux, DNS, TCP and socket tuning |
| [Routing, backends, and health](references/routing-backends-and-health.md) | Dynamic backends, balancing, retries, connection pools, health checks, SPOP |
| [HTTP policy and data handling](references/http-policy-and-data-handling.md) | Compression, filters, protocol defenses, fetches, converters, request actions |
| [Logging and observability](references/logging-and-observability.md) | Log profiles, traces, termination diagnostics, statistics, Prometheus |
| [Runtime API and Lua](references/runtime-api-and-lua.md) | Master CLI, runtime commands, certificate operations, Lua pattern and socket APIs |

## Upgrade blockers first

### Preserve an intentional balancing policy

Since 3.3.0, a backend without `balance` uses `random`, with a power-of-two
choice, instead of `roundrobin`. Make the old behavior explicit before an
upgrade:

```haproxy
backend application
    balance roundrobin
```

The random tie-breaker changed again in 3.4.0: equal connection counts are
resolved using recent HTTP request rates.

### Account for early client aborts

HTTP backends enable `option abortonclose` by default since 3.3.0. Explicitly
disable it only if an abandoned client request must still reach the server.
The option is also accepted in frontends.

### Remove duplicate names

Warnings introduced in 3.1.0 became configuration errors in 3.3.0. Keep names
unique across `frontend`, `listen`, `backend`, `defaults`, and `log-forward`
sections, and keep server names unique in their applicable scope.

### Migrate removed and deprecated directives

- `program` sections and legacy C mailers were removed in 3.3.0; use Lua
  mailers.
- Replace `dispatch <address>` with `server dispatch <address>` before 3.5.
- Replace `transparent` or `option transparent` with a server at `0.0.0.0`
  before 3.5.
- Replace `tune.quic.frontend.*` names with `tune.quic.fe.*`.
- Start master-worker mode with `-W` or `-Ws`, not the deprecated global
  `master-worker` directive.
- Replace `no-quic` with `tune.quic.listen on|off`.
- Replace shared `filter compression` plus direction selection with
  `filter comp-req` and `filter comp-res`; `compression-direction` is
  deprecated.
- Use `tune.idle-pool.shared`; `tune.takeover-other-tg-connections` is
  superseded and deprecated.
- Move OpenTracing integrations to the OpenTelemetry add-on before the planned
  OpenTracing removal in 3.5.

Use `expose-deprecated-directives` only as a temporary compatibility aid. Empty
configuration arguments warn in 3.2.0 and are intended to become errors; write
`${NAME[*]}` when an empty environment expansion is deliberate.

## High-value configuration changes

### Create a backend without reloading

Runtime-created backends in 3.4.0 are invisible to routing until published:

```text
add backend test-backend from mydefaults mode http
add server test-backend/server1 127.0.0.1:3000 check
enable server test-backend/server1
enable health test-backend/server1
publish backend test-backend
```

For removal, put servers in maintenance, wait for `srv-removable`, delete the
servers, unpublish the backend, wait for `be-removable`, then delete it. A
disabled or unpublished backend selected by `use_backend` or `default_backend`
is skipped unless `force-be-switch` is set.

### Reuse health-check definitions

Named `healthcheck` sections in 3.4.0 can contain a check type and its
`http-check` or `tcp-check` actions:

```haproxy
healthcheck ready_h2
    type httpchk
    http-check connect alpn h2
    http-check send meth HEAD uri /health ver HTTP/2 hdr Host www.example.com

backend application
    server app1 10.0.0.1:80 check healthcheck ready_h2
```

Use server `init-state` when traffic must wait for the first successful health
check. Use `check-reuse-pool` to run checks on idle pooled connections, and
`strict-maxconn` when a server limit applies to open TCP connections rather
than concurrent HTTP requests.

### Separate compression directions and order filters

```haproxy
backend application
    filter comp-res
    compression algo gzip
    compression type text/html text/plain application/json
```

`filter-sequence` controls execution independently of declaration order. A
declared filter omitted from the sequence does not run, which is useful when
ordering compression and bandwidth limiting but can silently disable a filter.

### Apply per-certificate frontend TLS policy

Use `ssl-f-use` to select a `crt-store` certificate and attach its TLS versions,
ALPN, ciphers, or signature algorithms independently of `bind`:

```haproxy
crt-store site_certs
    load crt "example.pem" alias "example"

frontend public_https
    bind :443 ssl
    ssl-f-use crt "@site_certs/example" ssl-min-ver TLSv1.2
```

Server-side TLS now derives SNI from the HTTP `host` header by default. Control
traffic with `sni-auto` or `no-sni-auto`, and checks with `check-sni-auto` or
`no-check-sni-auto`.

## Protocol and overload safeguards

- `quic-initial` rules can `accept`, `reject`, silently `dgram-drop`, or
  `send-retry` before a QUIC handshake finishes.
- `tune.h2.fe.max-frames-at-once`, `tune.h2.be.max-frames-at-once`, and
  `tune.h2.fe.max-rst-at-once` bound HTTP/2 work; RST limits from 1 to 10
  mitigate attacks but low values may increase interactive or gRPC latency.
- `tune.h2.fe.max-total-streams` retires long-lived incoming connections.
  `tune.streams-elasticity` reduces concurrency near `maxconn`, and
  `tune.h2.fe.max-concurrent-streams` supports `rq-load` and a `min` floor.
- HTTP/1 glitch thresholds are `tune.h1.fe.glitches-threshold` and
  `tune.h1.be.glitches-threshold`; graceful close starts at 75 percent of a
  terminating threshold.
- `tune.glitches.kill.cpu-usage` gates threshold termination by CPU usage. Its
  default `0` kills regardless of load; a nonzero value needs an H2 or QUIC
  glitch threshold.
- `tune.quic.fe.stream.max-total` sends HTTP/3 GOAWAY after a connection's
  lifetime request cap, then closes after active transfers finish.

## Diagnostics and observability

Use transaction-stage `log profile` definitions for destination-specific
formats at `accept`, `request`, `connect`, `response`, `close`, or `error`.
`do-log` emits extra records and, since 3.4.0, may select a profile per action:

```haproxy
http-request do-log profile syslog
```

Tracing is supported and controllable through the Runtime API. Select the
narrowest source (`h1`, `h2`, `h3`, `quic`, `qmux`, `ssl`, `acme`, `fcgi`,
`spop`, `peers`, or `check`) and stop it after diagnosis.

Add `term_events` to access logs when the full sequence of request termination
states matters. For selective error context, combine `bs.debug_str` or
`fs.debug_str` with `when(condition)` and inspect `last_entity` and
`waiting_entity`.

The Prometheus endpoint exposes `haproxy_sticktable_local_updates` per stick
table. `show stat typed` marks experimental shared-memory metrics `P` or `V`,
while `show info` reports map and ACL line additions and removals.

## Runtime and scripting cautions

- `@@<relative-pid>` keeps a Master CLI worker session interactive and carries
  prompt mode into the worker; `prompt` accepts `n`, `i`, and `p`.
- ACME certificates initially held only in memory must be persisted with
  `dump ssl cert`; `haproxy-dump-certs` can write socket-obtained certificates.
- `add ssl crt-list` accepts `crt-store` aliases without verifying that the
  filesystem path matches the in-memory name. The caller must validate it.
- Lua fetch booleans remain integer `0` and `1` unless
  `tune.lua.bool-sample-conversion normal` is configured.
- Use `core.get_patref` for mutable ACL or Map references, including atomic
  whole-file replacement through `prepare()` and `commit()`.
- `AppletTCP:receive()` accepts an optional timeout for periodic work.

## Operational checklist

1. Pin a feature branch and keep its bug-fix component current.
2. Check the branch support state and reproduce issues on its newest patch.
3. Run configuration validation with the deployment binary.
4. Review warnings for deprecated, empty, experimental, privilege, CPU, and
   thread-group configuration.
5. Make changed defaults explicit when old behavior is required.
6. Treat experimental ACME, ECH, shared statistics, backend HTTP/3, and QMux
   as opt-in features requiring `expose-experimental-directives`.
7. On reload, verify worker, socket, statistics, and dynamically created
   backend state according to the feature's persistence rules.
8. Use the topic references for exact directives, sample semantics, and
   migration details before editing production configuration.
