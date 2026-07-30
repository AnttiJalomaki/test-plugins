# Configuration and transports

## Listener and encrypted-service setup

DNS over QUIC is available when Unbound is built against libngtcp2 and a
QUIC-enabled OpenSSL (since 1.22.0):

```sh
./configure --with-libngtcp2=/path/to/ngtcp2 --with-ssl=/path/to/openssl
```

```conf
server:
    quic-port: 853
    quic-size: 8m
```

The statistics are `num.query.quic` and `mem.quic`. If the build lacks QUIC
support, Unbound ignores configured QUIC ports and warns when `quic-port` is
set. QUIC SSL setup occurs before chroot and privilege drop, and a QUIC
listening context is created only when needed (since 1.23.0).

DoT and DoH have separate SSL contexts, allowing different ALPN values. The
listener setup also avoids opening unencrypted channels alongside encrypted
channels on the same port (since 1.23.0). Listing HTTPS or QUIC ports in
`interface-automatic-ports` now initializes the corresponding protocol
(since 1.24.0).

## TLS version and connection selection

Use `tls-protocols` to select the supported TLS protocol versions (since
1.25.0). The prerelease `tls-use-system-versions` runtime option and
`--enable-system-tls` build option were removed in favor of this setting.

The 1.24.0 release disabled TLS 1.2, while 1.24.1 allowed it again. A
deployment requiring TLS 1.2 should not stay on 1.24.0.

For outgoing TLS, connection reuse is bound to the configured TLS name (since
1.25.0). A connection opened for one name is not reused for another name just
because both resolve to the same IP address.

## Per-zone forwarding transport

`forward-tcp-upstream` and `forward-tls-upstream` override the corresponding
global setting for one forward zone (since 1.22.0):

```conf
server:
    tcp-upstream: no
    tls-upstream: no

forward-zone:
    name: "."
    forward-tcp-upstream: yes
    forward-tls-upstream: yes
```

## Limits and timeout semantics

`max-global-quota` defaults to `200`, up from `128`, while keeping a bounded
amplification factor (since 1.23.0).

Loopback addresses are exempt from `wait-limit`. Both `wait-limit-netblock`
and `wait-limit-cookie-netblock` accept their two-argument forms (since
1.23.0).

The following limit semantics apply (since 1.24.0):

- `wait-limit: 0` disables every wait limit.
- `wait-limit-cookie: 0` can disable wait limits for cookie-validated clients.
- Exceeding a wait limit returns `SERVFAIL`.
- `discard-timeout` drops UDP queries, but it does not drop stream
  connections.

## Error replies and stream EOF

Malformed error cases receive error replies rather than silence, and the
replies do not reflect parts of the query (since 1.25.0). CHAOS-class queries
do not echo incoming EDNS extended RCODEs. EOF from a TCP client cancels
pending replies and closes the connection.

## Protocol record support

RESINFO RR type 261 is supported with the `LDNS_RR_TYPE_RESINFO` symbol and a
TXT-like representation (since 1.23.0).
