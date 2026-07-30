# Observability and Runtime APIs

## Logging by transaction stage

`log profile` in 3.1.0 assigns formats independently at `accept`, `request`,
`connect`, `response`, `close`, `error`, or `any`, and binds the profile to a
particular log destination. One profile may therefore emit at several stages
with destination-specific formats. The `do-log` action emits extra records as
traffic is processed.

Starting in 3.4.0, each `do-log` action can select its own profile instead of
all actions in a frontend sharing the profile chosen on the `log` line:

```haproxy
http-request do-log profile syslog
```

## Tracing

Tracing is supported rather than experimental starting in 3.1.0. It has a
dedicated configuration section and Runtime API controls. Trace sources include
`h1`, `h2`, `h3`, `quic`, `qmux`, `fcgi`, `spop`, `peers`, and `check`, allowing
subsystem-focused debugging.

The Runtime API `trace` command adds an `ssl` source for TLS events in 3.2.0.
An `acme` source for certificate automation arrives in 3.3.0:

```haproxy
traces
    trace acme sink stdout level user event +any verbosity clean start now
```

## Termination and rule diagnostics

### Conditional fields

Converter `when(condition)` in 3.1.0 returns its input unchanged when true and
no value otherwise. Use it to emit fields such as `bs.debug_str` and
`fs.debug_str` only for relevant failures.

Fetches `last_entity` and `waiting_entity` identify the operation interrupted by
a timeout or error. They can also identify the final evaluated rule leading to
an accept, redirect, or deny.

### Multi-event termination state

Fetch `term_events` in 3.2.0 returns a comma-separated sequence of request
termination states rather than only the final stream state. Append it to the
access format and decode it later with the supplied `term_events` program.

```haproxy
log-format "$HAPROXY_HTTP_LOG_FMT %[term_events]"
```

## Master CLI sessions and event rings

When selecting a worker by relative PID, use `@@` instead of `@` in 3.2.0 to
keep the Master CLI session interactive until explicit exit or command
completion. The worker inherits the master's prompt mode. The `prompt` command
supports `n`, `i`, and `p` modes.

Persistent sessions can subscribe to event rings. The `dpapi` ring was added
for this facility and initially carries ACME notifications.

## Persistent and operational statistics

Experimental shared-memory statistics in 3.3.0 require:

- global `expose-experimental-directives`;
- global `shm-stats-file`; and
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

A reload preserves these statistics, but a process restart does not. `show stat
typed` labels each metric `P` for persistent or `V` for volatile.

`show dev` in 3.3.0 reports thread-to-CPU bindings. `show info` reports counts
of lines added to and removed from map and ACL files, which can reveal
automation that only appends entries.

The Prometheus endpoint in 3.4.0 exports
`haproxy_sticktable_local_updates`, a cumulative gauge of local updates for
each configured stick table. Monitor its rate to understand update activity.

## Sample fetches

### Connection, TLS ClientHello, counters, and dates

The following arrive in 3.2.0:

- `bc_reused` reports whether the transfer reused a backend connection.
- `req.ssl_cipherlist`, `req.ssl_keyshare_groups`, `req.ssl_sigalgs`, and
  `req.ssl_supported_groups` expose binary TLS ClientHello capabilities.
- `sc_key(<ctr>)` returns the tracked-counter key.
- `table_clr_gpc(<idx>[,<table>])` mutates a general-purpose counter and returns
  its previous value.
- `table_inc_gpc(<idx>[,<table>])` mutates a general-purpose counter and returns
  its new value.
- `accept_date` and `request_date` fall back to the session date when no stream
  exists, including an early TLS-handshake failure.

### Directional byte counts

The 3.3.0 byte fetches have intentionally asymmetric viewpoints:

| Fetch | Meaning |
| --- | --- |
| `req.bytes_in` | Client-to-HAProxy bytes; alias of `bytes_in` |
| `req.bytes_out` | HAProxy-to-server bytes |
| `res.bytes_in` | Server-to-HAProxy bytes; alias of `bytes_out` |
| `res.bytes_out` | HAProxy-to-client bytes |

### HTTP versions, timeouts, TLS, and threads

As of 3.4.0, `req.ver` and `res.ver` consistently return `major.minor` for
HTTP/1, HTTP/2, and HTTP/3. `capture.req.ver` and `capture.res.ver` consistently
return `HTTP/major.minor`.

Also added in 3.4.0:

- `cur_connect_timeout`, `cur_queue_timeout`, and `cur_tarpit_timeout` return
  the active stream timeouts in milliseconds;
- `fe_tarpit_timeout` returns the configured frontend tarpit timeout;
- `ssl_fc_crtname` returns the selected incoming certificate name; and
- `tgroup` returns the zero-based position of the calling thread group.

## Data converters

Converters added or extended in 3.3.0 include:

- `base2`, which renders every input byte as eight binary digits;
- `le2dec`, which renders little-endian binary chunks as unsigned decimals;
- `aes_gcm_enc` and `aes_gcm_dec`, which accept an optional AAD argument for
  authenticated additional data.

Converters added in 3.4.0 include:

- `jwt_decrypt_cert`, `jwt_decrypt_secret`, and `jwt_decrypt_jwk`, which decrypt
  JWT input with a certificate, base64-encoded secret, or JSON Web Key;
- `aes_cbc_enc` and `aes_cbc_dec`, which process raw bytes with
  AES-128/192/256-CBC according to the `bits` argument; and
- `fe_exists`, which reports whether its input names a configured frontend.
