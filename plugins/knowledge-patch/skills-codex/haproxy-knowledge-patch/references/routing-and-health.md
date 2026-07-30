# Routing and Health Checks

## Backend modes and selection

### SPOP-native SPOE agents

SPOE runs as a mux as of 3.1.0, with native `mode spop` backends. These
backends support every load-balancing algorithm and can share idle connections
between threads. Existing SPOA agents remain compatible.

```haproxy
backend spoa_agents
    mode spop
    balance roundrobin
    server agent1 127.0.0.1:12345
```

### Default and tie-breaking balance behavior

Starting in 3.3.0, an ordinary backend with no explicit `balance` directive
uses `random` rather than `roundrobin`. It uses power-of-two choices: sample
two servers and select the less-loaded one. Configure `balance roundrobin` to
retain the old default.

As of 3.4.0, equal concurrent-connection counts are broken by comparing recent
HTTP request rates. Expect different distribution in large pools where many
servers previously looked equally loaded.

### Abort abandoned HTTP requests

Backends in `mode http` enable `option abortonclose` by default in 3.3.0. This
allows processing to stop before an abandoned client request is sent to a
server. The option is also valid in a frontend.

## Runtime-created backends

The Runtime API can create and remove complete backends without a reload as of
3.4.0. A newly added backend is unavailable to routing until it is published.
A disabled or unpublished backend selected by `use_backend` or
`default_backend` is skipped unless `force-be-switch` applies.

```text
add backend test-backend from mydefaults mode http
add server test-backend/server1 127.0.0.1:3000 check
enable server test-backend/server1
enable health test-backend/server1
publish backend test-backend
```

Remove a dynamic backend in this order:

1. Put every server in maintenance.
2. Wait for each server to report `srv-removable`, then delete it.
3. Unpublish the backend.
4. Wait for `be-removable`, then delete the backend.

Named `defaults` sections stay in memory so runtime creation can use them. If
the deployment never creates dynamic backends, global `tune.defaults.purge`
releases that retained memory.

## Retry and delay policy

### Choose retry counts per transaction

The `set-retries` action is valid in `tcp-request` and `http-request` rules as
of 3.1.0. Select the retry budget from the application path, method, or client:

```haproxy
http-request set-retries 0 if METH_POST
```

`retry-on` accepts HTTP status 421 as of 3.2.0, so a request misdirected to a
backend server that cannot serve it can be retried elsewhere.

### Pause traffic conditionally

The `pause` action in 3.2.0 delays request or response processing by a fixed
millisecond value or a sample expression. It can slow rate-limit offenders or
implement another bounded delay policy.

```haproxy
http-request pause 250
http-response pause 250
```

## Connection limits and compatibility

### Enforce an upstream connection cap

Server argument `strict-maxconn` in 3.2.0 makes `maxconn` count open TCP
connections rather than concurrent HTTP requests. Use it when the upstream has
a hard connection limit, especially when connection reuse otherwise decouples
request concurrency from socket count.

### Relax incomplete WebSocket handshakes deliberately

The existing backend directives `accept-unsafe-violations-in-http-request` and
`accept-unsafe-violations-in-http-response` also tolerate missing expected
WebSocket headers as of 3.2.0. They deliberately weaken parser enforcement;
scope their use to a known compatibility need.

## Health checks

### Set a Host header with legacy `option httpchk`

Since 3.1.0, `option httpchk` accepts a Host header directly. Do not encode the
header through fake strings in the `httpchk` line.

### Gate server readiness

The server `init-state` setting in 3.1.0 can keep a server down at startup, or
after it leaves maintenance, until its first successful health check.

### Reuse idle connections for checks

Server argument `check-reuse-pool` in 3.2.0 sends health checks over idle
pooled connections instead of always opening a new connection. This saves TCP
and TLS handshake work and supports reverse-HTTP permanent connections.

### Reuse named health-check definitions

A named `healthcheck` section in 3.4.0 can contain any supported check type and
its `http-check` or `tcp-check` actions. A server selects the definition with
the `healthcheck` argument. Definitions can be shared between backends while
different servers in one backend select different checks.

```haproxy
healthcheck mycheck
    type httpchk
    http-check connect alpn h2
    http-check send meth HEAD uri /health ver HTTP/2 hdr Host www.example.com

backend webservers
    server web1 10.0.0.1:80 check healthcheck mycheck
```
