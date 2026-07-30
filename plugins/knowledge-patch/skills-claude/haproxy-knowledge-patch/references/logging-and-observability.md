# Logging and Observability

## Transaction-stage log profiles

`log profile` introduced in 3.1.0 assigns formats independently at `accept`,
`request`, `connect`, `response`, `close`, `error`, or `any`. A profile is tied
to a particular log destination, so one transaction can emit at several stages
with destination-specific formats.

The `do-log` rule action emits additional logs while traffic is processed.
Since 3.4.0, each invocation can select a profile rather than every invocation
in a frontend sharing the profile selected by its `log` line:

```haproxy
http-request do-log profile syslog
```

Use stage-specific formats to avoid computing request-only fields during later
events, and make each destination's parsing contract explicit.

## Supported tracing

Tracing is supported rather than experimental since 3.1.0. It has a dedicated
configuration section and Runtime API controls. Focus collection by subsystem;
sources include `h1`, `h2`, `h3`, `quic`, `qmux`, `fcgi`, `spop`, `peers`, and
`check`.

The `ssl` source was added in 3.2.0 for TLS events. The `acme` source was added
in 3.3.0 for certificate automation:

```haproxy
traces
    trace acme sink stdout level user event +any verbosity clean start now
```

Enable the narrowest useful source, sink, event mask, and verbosity, then stop
the trace through the Runtime API after the incident to limit overhead and
data volume.

## Termination diagnostics

`term_events` added in 3.2.0 records a comma-separated sequence of request
termination states rather than only the final stream state. Add it to an access
log and decode it with the supplied `term_events` program:

```haproxy
log-format "$HAPROXY_HTTP_LOG_FMT %[term_events]"
```

The `when(condition)` converter can conditionally emit `bs.debug_str` and
`fs.debug_str`. `last_entity` and `waiting_entity` identify the operation
interrupted by a timeout or error, or the last evaluated rule that led to an
accept, redirect, or deny.

## Persistent shared-memory statistics

Experimental shared-memory statistics introduced in 3.3.0 persist across a
reload, but not a full process restart. They require all of:

- global `expose-experimental-directives`;
- a global `shm-stats-file`;
- a unique `guid` on every participating frontend, backend, and server.

```haproxy
global
    expose-experimental-directives
    shm-stats-file /dev/shm/haproxy-stats

frontend example
    guid a88e2a95-547e-47f1-b406-ea82ea47abcc
    bind :80
    use_backend webservers

backend webservers
    guid 3db38dc1-4aa8-4172-b7de-affb7f1f51a8
    server web1 172.16.0.12:80 check guid 775e29c2-0b97-4f19-9976-dba604b833f4
```

`show stat typed` marks each metric `P` for persistent or `V` for volatile.
Missing or reused GUIDs break the intended mapping across reloads.

## Runtime diagnostics

- `show dev` reports thread-to-CPU bindings since 3.3.0.
- `show info` reports added and removed line counts for Map and ACL files since
  3.3.0. A continually rising added count without removals can reveal faulty
  automation.
- `haproxy -vv` reports socket-owner and UDP GSO support, useful when
  diagnosing QUIC pacing or batching.
- `accept_date` and `request_date` fall back to the session date when an early
  failure, such as TLS handshake failure, occurs before a stream exists.

## Stick-table update telemetry

The Prometheus endpoint exports `haproxy_sticktable_local_updates` since 3.4.0.
It is a cumulative gauge for local updates on each configured stick table.
Graph its rate to observe write activity; do not interpret the raw cumulative
value as a per-second rate.

## HTTP/2 error-log scope

Global `tune.h2.log-errors` introduced in 3.4.0 selects stream-scope logging,
connection-only logging, or no HTTP/2 error logging. Default `stream` is the
most verbose mode and includes stream detail. Reduce it deliberately when log
volume outweighs per-stream diagnosis.

## Stats-page version disclosure

The Stats page no longer displays the HAProxy version by default since 3.4.0.
Set `stats show-version` to opt back in. Consider whether exposing the exact
version is appropriate for the page's audience.

## OpenTelemetry transition

OpenTelemetry is available as an add-on replacing OpenTracing. The OpenTracing
filter is officially deprecated in 3.4.0 and remains scheduled for removal in
3.5. Plan instrumentation migration before upgrading to the removal release.
