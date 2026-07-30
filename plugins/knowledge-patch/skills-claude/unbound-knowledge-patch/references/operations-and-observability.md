# Operations and observability

## Dnstap

### Sampling

Since 1.21.0, `dnstap-sample-rate` limits high-volume output to one of every
N messages:

```conf
dnstap:
    dnstap-sample-rate: 100
```

### Fast-reload propagation

Since 1.24.0, dnstap configuration changes are copied from the daemon to
worker threads after `unbound-control fast_reload`.

## DNS Error Reporting

Since 1.23.0, RFC 9567 DNS Error Reporting is available through
`dns-error-reporting`. Sent reports are counted by
`num.dns_error_reports`.

## Reloads

### Fast reload

Since 1.23.0:

```sh
unbound-control fast_reload
```

The command reads changed configuration in a thread and pauses service
threads only briefly, keeping DNS interruption below a second.

### Redis outage handling

A reload since 1.23.0 no longer fails merely because the Redis cachedb backend
cannot connect or respond. The failure is logged instead.

### TLS and selected iterator settings

Since 1.25.0, reloads rebuild affected TLS contexts when certificate files
change. `fast_reload` handles `tls-service-key`, `tls-service-pem`, and
`tls-cert-bundle`, and propagates changes to `iter-scrub-ns`,
`iter-scrub-cname`, and `max-global-quota`.

## Cache control

### Negative cache flush reporting

Since 1.23.0, `unbound-control flush_negative` correctly reports removed
data.

### Targeted lookup

Since 1.24.0, inspect selected names with:

```sh
unbound-control cache_lookup example.com
unbound-control cache_lookup +t .
```

The `+t` form accepts TLD and root names. Matching subnet-cache contents are
included.

### Responsive dumps

Since 1.24.0, `unbound-control dump_cache` periodically releases cache locks
and decouples file-descriptor activity from cache lookups. This keeps the
server responsive during long dumps.

## Wait limits, timeouts, and quotas

### Limit forms and loopback exemption

Since 1.23.0, loopback addresses are exempt from `wait-limit`.
`wait-limit-netblock` and `wait-limit-cookie-netblock` accept their
two-argument forms. Statistics expose wait-limit and discard-timeout
activity.

`max-global-quota` defaults to `200` since 1.23.0 instead of `128`, while the
amplification factor remains bounded.

### Zero and rejection behavior

Since 1.24.0:

- `wait-limit: 0` disables all wait limits.
- `wait-limit-cookie: 0` can disable COOKIE-validated wait limits.
- Exceeding a wait limit returns `SERVFAIL`.
- `discard-timeout` drops UDP queries, but not stream connections.

### Mesh accounting

Since 1.24.0, statistics expose `num.queries.replyaddr_limit` and
`requestlist.current.replies`. Packets dropped by `discard-timeout` reduce
the mesh reply-address-in-use count as well.

## Remote-control behavior

### Control key access

Since 1.23.0, members of the `unbound` group have access to the control key.

### Strict argument checking

Since 1.24.0, `unbound-control` commands that take no arguments reject
extraneous arguments.

### Per-interface ports

Since 1.25.0, a control listener can specify its own port:

```conf
remote-control:
    control-interface: 127.0.0.1@8953
```

## Diagnostics

### Socket-buffer warning

Since 1.24.0, Unbound warns when the operating system does not grant the
`so-sndbuf` `setsockopt` request.

### Linux thread IDs

Since 1.25.0, Linux logging can use the system-wide thread ID rather than
Unbound's internal counter, improving correlation with system tools:

```conf
server:
    log-thread-id: yes
```
