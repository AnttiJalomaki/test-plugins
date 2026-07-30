---
name: unbound-knowledge-patch
description: Unbound
version: 1.25.0
license: MIT
metadata:
  author: Nevaberry
---


# Unbound

Use this skill when configuring, upgrading, operating, or troubleshooting
Unbound. Start with the breaking defaults and security-sensitive behavior
below, then open the topic reference that matches the task.

## Reference index

| Reference | Topics |
| --- | --- |
| [configuration-and-transports.md](references/configuration-and-transports.md) | Listener setup, DNS over QUIC, TLS selection, forwarding transports, limits, and protocol behavior |
| [cache-resolution-and-validation.md](references/cache-resolution-and-validation.md) | Cachedb, Redis, serve-expired, TTL handling, prefetch, DNSSEC, ECS, NAT64, and alias processing |
| [policy-and-auth-zones.md](references/policy-and-auth-zones.md) | RPZ, response-IP, local zones, auth zones, forwarding checks, notifications, and rebinding protection |
| [operations-and-observability.md](references/operations-and-observability.md) | Reloads, remote control, cache inspection, dnstap, error reporting, statistics, rate limits, and logging |
| [platform-and-build.md](references/platform-and-build.md) | Build dependencies, service units, platform behavior, root keys, and reproducible builds |

## Breaking defaults and upgrade checks

### Configure subnet caching explicitly

Enabling subnet support at build time no longer inserts `subnetcache` into
`module-config`. Add it explicitly when EDNS Client Subnet caching is required:

```conf
server:
    module-config: "subnetcache validator iterator"
```

Check custom processing orders separately. For DNS64 answers subject to
response-IP and RPZ policy, use:

```conf
server:
    module-config: "respip dns64 validator iterator"
```

Do not assume that placing cachedb in
`"respip dns64 validator cachedb iterator"` works.

### Review serve-expired behavior

The defaults are now:

```conf
server:
    serve-expired-ttl: 86400
    serve-expired-client-timeout: 1800
```

`serve-expired-reply-ttl` cannot exceed the original record TTL. Records leave
cache at TTL 0, and upstream TTL-0 answers are not stored in cachedb.

### Account for newly local names

`resolver.arpa` and `service.arpa` are served locally by default. Check
forwarding expectations for these names during an upgrade.

### Recheck resource limits

`max-global-quota` defaults to `200`. Loopback clients are exempt from
`wait-limit`; setting `wait-limit: 0` disables all wait limits, and
`wait-limit-cookie: 0` disables the cookie-validated limit. A wait-limit
excess returns `SERVFAIL`. `discard-timeout` drops UDP queries, not stream
connections.

### Select TLS versions explicitly

Use `tls-protocols` to choose supported TLS versions. Do not use the removed
`tls-use-system-versions` or `--enable-system-tls` controls. If a deployment
must run the release that disabled TLS 1.2, move to a patch release that
restores TLS 1.2 compatibility.

## Security-sensitive configuration

### Bound signature-set work

`iter-scrub-rrsig` limits the number of RRSIG records retained by the iterator
scrubber and defaults to `8`:

```conf
server:
    iter-scrub-rrsig: 8
```

### Preserve rebinding protection

`private-address` filtering applies to SVCB and HTTPS records as well as
address records. Keep private ranges complete when relying on this control.

### Harden glue consistently

`harden-unverified-glue` also controls missing-AAAA lookups made during cache
fill. Enabling it therefore hardens both ordinary and cache-fill glue paths.

### Understand alias validation

DNSSEC handling accepts YXDOMAIN only with DNAME, rejects signatures made by
revoked DNSKEYs, and applies stricter trust checks to DNAME-to-CNAME and
wildcard-CNAME chains.

### Rotate EDNS COOKIE secrets

Persist rollover state with `cookie-secret-file`, then use remote control to
add, activate, inspect, and drop secrets:

```conf
server:
    cookie-secret-file: "unbound_cookiesecrets.txt"
```

```sh
unbound-control add_cookie_secret SECRET
unbound-control activate_cookie_secret SECRET
unbound-control print_cookie_secrets
unbound-control drop_cookie_secret SECRET
```

`ip-ratelimit-cookie` is enforced for COOKIE-bearing clients. Recheck existing
settings after an upgrade because a previously configured value may now take
effect.

## Transport quick reference

### Enable DNS over QUIC

Build with libngtcp2 and a QUIC-capable OpenSSL, then configure the listener
and memory allowance:

```sh
./configure --with-libngtcp2=/path/to/ngtcp2 --with-ssl=/path/to/openssl
```

```conf
server:
    quic-port: 853
    quic-size: 8m
```

Confirm `num.query.quic` and `mem.quic` in statistics. A build without QUIC
support ignores QUIC ports and warns when `quic-port` is configured.

### Override transport per forward zone

Zone-level settings can override global upstream transport:

```conf
server:
    tcp-upstream: no
    tls-upstream: no

forward-zone:
    name: "."
    forward-tcp-upstream: yes
    forward-tls-upstream: yes
```

An upstream TLS connection is reusable only when its TLS name matches the new
destination, even if both destinations resolve to the same address.

### Keep encrypted listeners distinct

DoT and DoH use separate SSL contexts and can advertise different ALPN values.
Unbound avoids opening an unencrypted channel beside an encrypted channel on
the same port. Adding HTTPS or QUIC ports to `interface-automatic-ports`
initializes the corresponding protocol.

## Operations quick reference

### Reload with a short interruption

```sh
unbound-control fast_reload
```

`fast_reload` parses changed configuration in a separate thread and briefly
pauses service threads. It propagates dnstap changes, reloads changed service
keys, certificates, and trust bundles, and applies changes to
`iter-scrub-ns`, `iter-scrub-cname`, and `max-global-quota`.

A reload logs Redis backend failures instead of failing solely because Redis
is unavailable.

### Inspect selected cache entries

```sh
unbound-control cache_lookup example.com
unbound-control cache_lookup +t .
```

`cache_lookup` prints cached RRsets and messages and includes matching subnet
cache data. Use `+t` for TLD and root names. Long `dump_cache` operations
periodically release locks so the resolver remains responsive.

### Select remote-control listener ports

```conf
remote-control:
    control-interface: 127.0.0.1@8953
```

Commands that take no arguments reject extra arguments. Members of the
`unbound` group can access the control key.

### Sample high-volume dnstap traffic

```conf
dnstap:
    dnstap-sample-rate: 100
```

The value emits one out of every N messages. Confirm worker configuration
after a fast reload when dnstap settings change.

### Correlate Linux worker logs

```conf
server:
    log-thread-id: yes
```

This selects the system-wide Linux thread ID instead of Unbound's internal
counter, making correlation with operating-system tools easier.

## Validation checklist

- Run `unbound-checkconf` and investigate ineffective `nodefault` warnings.
- Check warnings for hostname-based stub or forward servers; they may create a
  circular dependency on the resolver itself.
- Confirm the operating system granted the requested `so-sndbuf`.
- Verify QUIC build support before relying on configured QUIC ports.
- Confirm the processing order for subnet cache, response-IP, DNS64, and
  validation features.
- Exercise certificate renewal with the chosen reload path.
- Test TTL-0, serve-expired, and external cachedb behavior independently.
- Check local-zone behavior for DS queries and dynamically added blocking
  zones.
