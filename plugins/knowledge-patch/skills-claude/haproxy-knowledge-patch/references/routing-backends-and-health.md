# Routing, Backends, and Health

## Runtime-created backends

The Runtime API can create, publish, unpublish, and delete complete backends
without a reload since 3.4.0. Creation and publication are distinct: routing
cannot use a backend until it is published.

```text
add backend test-backend from mydefaults mode http
add server test-backend/server1 127.0.0.1:3000 check
enable server test-backend/server1
enable health test-backend/server1
publish backend test-backend
```

A disabled or unpublished backend chosen by `use_backend` or
`default_backend` is skipped unless `force-be-switch` is set.

Safe deletion is a staged drain:

1. Put every server into maintenance.
2. Wait for each server's `srv-removable` state.
3. Delete each server.
4. Unpublish the backend.
5. Wait for `be-removable`.
6. Delete the backend.

Named `defaults` sections stay in memory to seed dynamic creation. Set global
`tune.defaults.purge` when dynamic backends are not used and this retained
state should be released.

## Balancing behavior

A backend with no explicit `balance` uses `random` since 3.3.0 rather than
`roundrobin`. Random balancing uses a power-of-two choice: sample two servers
and choose the less-loaded one. Configure `balance roundrobin` when order is
required.

Since 3.4.0, two random candidates with equal concurrent-connection counts are
compared by recent HTTP request rate. This affects large pools in which many
servers otherwise appear equally loaded.

SPOP-native backends introduced in 3.1.0 use `mode spop`. SPOE is implemented
as a mux, these backends accept any load-balancing algorithm, and idle
connections can be shared across threads. Existing SPOA agents stay compatible:

```haproxy
backend spoa_agents
    mode spop
    balance roundrobin
    server agent1 127.0.0.1:12345
```

## Retries and pacing policies

`set-retries` is available in `tcp-request` and `http-request` rules since
3.1.0. It chooses the retry count per request from a literal or evaluated
policy:

```haproxy
http-request set-retries 0 if METH_POST
```

`retry-on` accepts status 421 since 3.2.0, allowing a request misdirected to a
backend server that cannot serve it to be retried elsewhere.

The `pause` action delays request or response processing for a fixed number of
milliseconds or a sample expression (since 3.2.0). It can slow rate-limit
offenders, but it deliberately occupies transaction time:

```haproxy
http-request pause 250
http-response pause 250
```

## Connection limits and idle pools

Server `strict-maxconn` makes `maxconn` count open TCP connections rather than
concurrent HTTP requests (since 3.2.0). Use it for upstreams with a hard socket
limit.

Server `check-reuse-pool` performs health checks over idle pooled connections
instead of opening a new connection. This reduces connection and TLS handshake
cost and supports reverse-HTTP permanent connections.

Global `tune.idle-pool.shared` introduced in 3.4.0 controls cross-thread idle
server connection sharing:

- `on`: share inside a thread group.
- `full`: share across all threads.
- `off`: disable sharing, primarily for debugging.

It supersedes and deprecates `tune.takeover-other-tg-connections`.

## Health-check definitions

### Startup gating

Server `init-state` introduced in 3.1.0 can keep a server down at startup or
after maintenance until its first health check succeeds. Use it where an
optimistic initial state could send traffic too early.

### Host in legacy HTTP checks

`option httpchk` accepts a Host header directly since 3.1.0. Do not encode it
through the older fake-string workaround in the `httpchk` line.

### Reusable sections

A named `healthcheck` section introduced in 3.4.0 can contain any supported
check type and its `http-check` or `tcp-check` actions. Servers select one with
the `healthcheck` argument, so servers in the same backend can use different
checks and definitions can be reused across backends:

```haproxy
healthcheck mycheck
    type httpchk
    http-check connect alpn h2
    http-check send meth HEAD uri /health ver HTTP/2 hdr Host www.example.com

backend webservers
    server web1 10.0.0.1:80 check healthcheck mycheck
```

## Backend TLS transports

Server-side TLS derives SNI from the HTTP `host` header automatically since
3.3.0. `sni-auto` and `no-sni-auto` govern normal traffic;
`check-sni-auto` and `no-check-sni-auto` govern checks.

Experimental backend HTTP/3 uses a `quic4@` address and normal backend TLS
verification. QMux in 3.4.0 instead carries HTTP/3/QUIC frames over TCP between
HAProxy endpoints with `alpn h3`. See the networking reference for complete
experimental enablement and tuning.

## Client-abort behavior

Since 3.3.0, HTTP backends enable `option abortonclose` by default. HAProxy can
stop work before forwarding an abandoned client request to a server. The
option is newly valid in frontends as well; make the desired behavior explicit
for workflows that must complete after client disconnect.
