# Operations and observability

## Fast and ordinary reloads

`unbound-control fast_reload` parses changed configuration in a thread and
briefly pauses service threads, keeping DNS interruption below one second
(since 1.23.0):

```sh
unbound-control fast_reload
```

Dnstap configuration is copied from the daemon to worker threads during fast
reload (since 1.24.0).

Certificate-file changes are detected during reload and TLS contexts are
rebuilt for DoT, DoH, DoQ, and outgoing DoT (since 1.25.0). `fast_reload`
handles `tls-service-key`, `tls-service-pem`, and `tls-cert-bundle`. It also
propagates changes to `iter-scrub-ns`, `iter-scrub-cname`, and
`max-global-quota`, allowing these updates and certificate renewals without a
full restart.

Reload does not fail solely because the Redis cachedb backend cannot connect
or respond (since 1.23.0); the backend details are in
[cache-resolution-and-validation.md](cache-resolution-and-validation.md).

## Cache inspection

`cache_lookup` prints cached RRsets and messages for selected names and
includes matching subnet-cache content (since 1.24.0):

```sh
unbound-control cache_lookup example.com
unbound-control cache_lookup +t .
```

Use `+t` to accept TLD and root names. `dump_cache` periodically releases cache
locks and separates file-descriptor activity from cache lookups, keeping the
resolver responsive during long dumps (since 1.24.0).

## Remote-control access and validation

Members of the `unbound` group can access the control key (since 1.23.0).

Each `control-interface` can name its own port with `IP@port` (since 1.25.0):

```conf
remote-control:
    control-interface: 127.0.0.1@8953
```

Commands that take no arguments reject extraneous arguments (since 1.24.0).

## EDNS COOKIE rotation and rate limiting

`cookie-secret-file` persists COOKIE secrets for rollover (since 1.21.0):

```conf
server:
    cookie-secret-file: "unbound_cookiesecrets.txt"
```

Use `add_cookie_secret`, `activate_cookie_secret`, and `drop_cookie_secret` to
rotate secrets. `print_cookie_secrets` displays the values currently in use.

`ip-ratelimit-cookie` is applied to COOKIE clients (since 1.21.0). An existing
configuration that already sets it begins enforcing that intended limit after
upgrade.

## Dnstap

`dnstap-sample-rate` emits one out of every N messages, reducing output volume
(since 1.21.0):

```conf
dnstap:
    dnstap-sample-rate: 100
```

Both dnstap and `unbound-dnstap-socket` can be linked in a build that does not
link OpenSSL (since 1.21.0).

## DNS Error Reporting

RFC 9567 DNS Error Reporting is enabled with `dns-error-reporting`, and sent
reports are counted in `num.dns_error_reports` (since 1.23.0).

## Limit and mesh statistics

Statistics report discard-timeout and wait-limit activity (since 1.23.0).
They also expose `num.queries.replyaddr_limit` and
`requestlist.current.replies` (since 1.24.0). Packets dropped by
`discard-timeout` decrement the mesh reply-address-in-use count.

## Diagnostics and logging

Unbound warns when the operating system does not grant the `so-sndbuf`
`setsockopt` request (since 1.24.0).

On Linux, `log-thread-id` selects the system-wide thread ID instead of
Unbound's internal counter (since 1.25.0):

```conf
server:
    log-thread-id: yes
```

This makes resolver messages easier to correlate with operating-system
diagnostic tools.
