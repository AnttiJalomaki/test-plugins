# Networking and protocols

## DNS

### Per-call resolver configuration

Since 28.1, option-list variants allow direct resolver calls to override
settings without changing shared configuration:

- `inet_res:gethostbyname/4`
- `inet_res:getbyname/4`
- `inet_res:gethostbyaddr/3`

### TSIG and error atoms

The internal `inet_dns_tsig` and `inet_res` modules in 28.1 verify the correct
TSIG timestamp. Their two undocumented DNS error atoms were corrected to the
RFC names `notauth` and `notzone`; update code that matches the former
incorrect atoms.

## TCP and sockets

### Keepalive and user timeout

Since 28.3, `gen_tcp` supports `TCP_KEEPCNT`, `TCP_KEEPIDLE`, and
`TCP_KEEPINTVL`. `TCP_USER_TIMEOUT` is supported by both `gen_tcp` and
`socket`.

### Batched messages

The socket implementation adds `recvmmsg()` and `sendmmsg()` operations in
29.0 to receive or send multiple messages per call.

### Accepted-socket inheritance

Since 29.0.2, the `gen_tcp_socket` accept path inherits options in the same
way as classic `gen_tcp`.

### SCTP peeloff

Since 29.0.1, an IPv6 SCTP socket returned by peeloff inherits the parent
socket's options.

## HTTP client

### Bounded `Retry-After` handling

Starting in 28.4, `httpc:request/4,5` retries once by default after a
`Retry-After` response and then returns the error response instead of
retrying indefinitely. The HTTP option `{autoretry, timeout()}` controls the
behavior. Configure it or implement an application retry policy when the
default is unsuitable.

```erlang
HttpOptions = [{autoretry, RetryTimeout}].
```

### Cross-origin redirects

Since 29.0.2, when `httpc` follows a redirect whose host or port changes, it
removes these headers:

- authorization
- proxy-authorization
- cookie
- referer
- origin

A client that intentionally forwards credentials across the boundary must
explicitly authorize the new target.

## FTP

Since 29.0.2, FTP clients reject passive-mode responses that redirect the
data connection to an arbitrary host instead of following the supplied
address.

## TLS distribution

Since 29.0.2, Erlang distribution over TLS enforces the same-LAN restriction
when `check_ip` is enabled. Connections accepted by older patch levels may be
rejected. Verify the intended LAN boundary and certificate setup during
rollout.

## SNMP

Since 29.0.1, `snmpm_usm:generate_outgoing_msg/5` no longer crashes with
`badmatch` while building an error response for an unknown user or engine ID.

