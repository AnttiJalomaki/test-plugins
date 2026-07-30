---
name: haproxy-knowledge-patch
description: HAProxy
version: 3.4.0
license: MIT
metadata:
  author: Nevaberry
---


# HAProxy Knowledge Patch

Use this skill when writing, reviewing, upgrading, or operating HAProxy
configuration, Runtime API automation, Lua extensions, or build and maintenance
procedures. Check the deployed feature branch and patch release before applying
version-dependent syntax. Treat live configuration validation, Runtime API
output, and current process behavior as authoritative.

## Reference index

| Reference | Topics |
| --- | --- |
| [upgrades-and-maintenance.md](references/upgrades-and-maintenance.md) | Breaking defaults, deprecated syntax, startup checks, branch and patch maintenance |
| [routing-and-health.md](references/routing-and-health.md) | Backend selection, retries, dynamic backends, balancing, health checks, SPOP |
| [tls-and-certificates.md](references/tls-and-certificates.md) | `crt-store`, frontend and backend TLS, ACME, ECH, passphrases, certificate utilities |
| [quic-http-and-networking.md](references/quic-http-and-networking.md) | QUIC, HTTP/1/2/3, QMux, DNS families, TCP controls, protocol defenses |
| [observability-and-runtime.md](references/observability-and-runtime.md) | Log profiles, tracing, Runtime and Master CLI, diagnostics, fetches, metrics |
| [filters-lua-and-performance.md](references/filters-lua-and-performance.md) | Compression, filter ordering, Lua APIs, CPU placement, buffers, connection pools |

## Upgrade hazards first

### Preserve backend behavior explicitly

HAProxy 3.3 changes a backend with no `balance` directive from `roundrobin` to
the power-of-two `random` policy. Declare the intended policy during an
upgrade:

```haproxy
backend application
    balance roundrobin
```

HTTP backends also enable `option abortonclose` by default in 3.3. Confirm that
abandoned client requests should stop before being sent upstream; explicitly
disable the option if application semantics require the old behavior.

### Remove duplicate and ambiguous configuration

Names duplicated across `frontend`, `listen`, `backend`, `defaults`, and
`log-forward` families, or duplicate server names, warn in 3.1 and fail in
3.3. Make every such name unique before upgrading.

An ACL may no longer contain more than one match type after `-m`; ambiguous
forms such as `path_beg -m reg` also warn. Rewrite each ACL with one explicit,
coherent match method.

Empty configuration arguments warn in 3.2 and are scheduled to fail in the
next version. For a deliberately empty environment expansion inside double
quotes, use `${NAME[*]}`.

### Migrate deprecated dispatch forms

Replace `dispatch <address>` before its planned 3.5 removal with a normal
server named `dispatch`. Give any retained legacy servers weight zero when
that is necessary to preserve dispatch behavior.

```haproxy
backend legacy_dispatch
    server dispatch 192.0.2.10:8080
```

Replace `transparent` or `option transparent` with a zero-address server to
retain routing to the original TPROXY destination:

```haproxy
backend original_destination
    server tproxy 0.0.0.0
```

Also replace:

- `tune.quic.frontend.*` with `tune.quic.fe.*`;
- global `master-worker` with command-line `-W` or `-Ws`;
- global `no-quic` with `tune.quic.listen on|off`;
- `compression-direction` and the shared compression filter with the split
  request or response filter;
- `tune.takeover-other-tg-connections` with `tune.idle-pool.shared`.

`program` sections and legacy C mailers were deprecated for removal in 3.3;
use Lua mailers. OpenTracing is deprecated and scheduled for removal in 3.5;
use the OpenTelemetry add-on.

### Recheck platform and automatic defaults

The `linux-glibc` build target requires Linux 4.17 starting in 3.3 because of
Kernel TLS. In the same version, `cpu-policy` defaults to `performance`,
automatic placement uses all available cores and NUMA nodes, and the former
64-thread automatic-placement limit is gone. Validate affinity and capacity
on heterogeneous and multi-NUMA hosts.

`dns-accept-family` defaults to `auto` in 3.3. It always enables IPv4 and
enables IPv6 only while the recurring connectivity probe succeeds.

## High-value configuration patterns

### Create a backend at runtime

A runtime-created backend is skipped by routing until published. Create it,
add and ready its servers, then publish it:

```text
add backend test-backend from mydefaults mode http
add server test-backend/server1 127.0.0.1:3000 check
enable server test-backend/server1
enable health test-backend/server1
publish backend test-backend
```

For removal, put every server into maintenance, wait for `srv-removable`,
delete the servers, unpublish the backend, wait for `be-removable`, then delete
the backend. Keep named `defaults` sections in memory for this workflow; set
`tune.defaults.purge` only when dynamic creation is unused.

### Reuse health checks

A named `healthcheck` section can contain a check type and its `http-check` or
`tcp-check` actions. Select it per server with the `healthcheck` argument:

```haproxy
healthcheck ready
    type httpchk
    http-check connect alpn h2
    http-check send meth HEAD uri /health ver HTTP/2 hdr Host example.com

backend application
    server app1 10.0.0.1:80 check healthcheck ready
```

Use `init-state` when a server must remain down after startup or maintenance
until its first health check succeeds. Use `check-reuse-pool` when checks may
reuse idle pooled connections.

### Apply certificate-specific frontend policy

Reference a `crt-store` entry with `ssl-f-use` to set certificate-specific TLS
versions, ALPN, ciphers, or signature algorithms without a crt-list:

```haproxy
crt-store certificates
    load crt "foo.com.crt" key "foo.com.key" alias "foo"

frontend public
    bind :443 ssl
    ssl-f-use crt "@certificates/foo" ssl-min-ver TLSv1.2
```

Backend TLS derives SNI from the HTTP `host` header in 3.3. Control traffic
with `sni-auto` or `no-sni-auto`, and checks with `check-sni-auto` or
`no-check-sni-auto`.

### Order and split filters

Use `filter comp-req` and `filter comp-res` for request and response
compression. `filter-sequence` defines execution order independently of
declaration order, and omitting a configured filter from the sequence disables
it.

```haproxy
backend application
    filter comp-res
    compression algo gzip
    compression type text/html text/plain application/json
```

This matters when compression and bandwidth limiting must run in a deliberate
order.

## Diagnostics and safety controls

### Log at the transaction stage that matters

`log profile` can assign destination-specific formats at `accept`, `request`,
`connect`, `response`, `close`, `error`, or `any`. `do-log` emits an additional
record while rules run, and in 3.4 each action can choose its profile:

```haproxy
http-request do-log profile syslog
```

Add `term_events` to access logs to retain the sequence of termination states,
not only the final stream state. Use `when(condition)` to include expensive
diagnostics such as `bs.debug_str` and `fs.debug_str` only when relevant.

### Use supported tracing narrowly

The `trace` Runtime API command controls supported trace sources for HTTP,
QUIC, QMux, TLS, ACME, SPOE/SPOP, peers, checks, and related subsystems. Enable
only the source and event scope needed for the incident, then disable it.

### Bound protocol abuse

For HTTP/2, set frontend and backend frame-batch limits, cap simultaneous
RST_STREAM processing, and optionally recycle connections after a lifetime
stream limit. Very low reset limits can add latency to interactive or gRPC
traffic.

HTTP/1 glitch thresholds are available on both sides. Threshold-based
termination begins graceful close at 75% of the configured threshold. A
CPU-gated kill policy can defer enforcement until configured CPU pressure, but
its default `0` enforces the threshold regardless of load.

## Maintenance decisions

Choose the feature branch and patch level independently. Even-numbered feature
branches are normally five-year LTS lines; odd-numbered branches are stable
lines maintained for roughly 12–18 months. Within a maintained branch, keep
the final bug-fix component current and reproduce problems on that release
before reporting them.

Interpret queued maintenance fixes separately from later development-branch
candidates. `MAJOR` and `CRITICAL` issues demand urgent action; `MEDIUM` issues
normally warrant an update or temporarily disabling the feature, while a
`MINOR` issue rarely justifies an update by itself.
