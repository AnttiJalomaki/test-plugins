---
name: unbound-knowledge-patch
description: Unbound
version: 1.25.0
license: MIT
metadata:
  author: Nevaberry
---


# Unbound Knowledge Patch

Load this skill when configuring, upgrading, operating, packaging, or
troubleshooting the Unbound validating resolver. It is especially useful for
changes involving encrypted DNS, cache behavior, response policy, DNSSEC,
remote control, cachedb, or service confinement.

## How to use this skill

1. Determine the deployed Unbound version and build features before applying
   version-sensitive advice.
2. Inspect `unbound-checkconf -o module-config` and the active listener,
   forwarding, cache, and policy configuration.
3. Read the topic reference that matches the task.
4. Apply defaults and compatibility changes deliberately during upgrades.
5. Validate configuration, reload behavior, statistics, and representative DNS
   answers after the change.

## Reference index

| Reference | Topics |
|---|---|
| [security-and-policy.md](references/security-and-policy.md) | DNSSEC, COOKIE controls, RPZ, local-zone policy, rebinding protection |
| [transports-and-tls.md](references/transports-and-tls.md) | DoQ, DoT, DoH, upstream TLS, listener activation, TLS reloads |
| [cache-and-recursion.md](references/cache-and-recursion.md) | cachedb, Redis, serve-expired, TTLs, ECS, DNAME, NAT64 |
| [zones-and-resolution.md](references/zones-and-resolution.md) | Forward and stub zones, auth-zones, subnet cache, DNS64, RESINFO |
| [operations-and-observability.md](references/operations-and-observability.md) | Remote control, fast reload, cache inspection, dnstap, limits, logging |
| [platforms-and-builds.md](references/platforms-and-builds.md) | Build flags, service units, Windows, BSD PF, Linux, QNX |

## Upgrade-critical changes

### Configure the subnet cache module explicitly

Building with subnet support no longer inserts `subnetcache` into
`module-config`. Add it explicitly when EDNS Client Subnet processing is
required:

```conf
server:
    module-config: "subnetcache validator iterator"
```

Check the configured order rather than assuming the build option changes it.

### Review changed defaults

- `serve-expired-ttl` defaults to `86400`.
- `serve-expired-client-timeout` defaults to `1800`.
- `max-global-quota` defaults to `200`.
- `resolver.arpa` and `service.arpa` are served locally by default.
- `module-config` defaults to `"validator iterator"`.

These changes can alter stale-answer behavior, resource limits, forwarding, and
module execution without a syntax change in an existing configuration.

### Select TLS versions explicitly

Use `tls-protocols` to choose supported protocol versions. Do not use the
pre-release `tls-use-system-versions` or `--enable-system-tls` controls.
Unbound 1.24.0 disabled TLS 1.2, while 1.24.1 permitted it again, so a
deployment that needs TLS 1.2 should not remain on 1.24.0.

### Recheck limit semantics

`wait-limit: 0` disables all wait limits, and `wait-limit-cookie: 0` can
disable limits for COOKIE-validated clients. A wait-limit rejection returns
`SERVFAIL`. `discard-timeout` drops UDP queries, not stream connections.
Loopback clients are exempt from `wait-limit`.

### Account for stricter cache exclusions

`forward-no-cache` and `stub-no-cache` now prevent both lookup and storage in
external cachedb, including applicable ECS paths. TTL-0 upstream answers are
not stored by cachedb, and DNAME or synthesized-CNAME data follows cache TTL
policy.

## Security quick reference

### Protect against oversized signature sets

`iter-scrub-rrsig` caps retained RRSIG records and defaults to 8:

```conf
server:
    iter-scrub-rrsig: 8
```

### Apply private-address filtering to modern service records

`private-address` filtering covers SVCB and HTTPS records as well as address
records. Keep the configured private ranges aligned with the network boundary
to prevent rebinding through service-binding answers.

### Rotate EDNS COOKIE secrets without restart

Persist rollover state with `cookie-secret-file`, then use
`add_cookie_secret`, `activate_cookie_secret`, and `drop_cookie_secret`.
Inspect active values with `print_cookie_secrets`.

```conf
server:
    cookie-secret-file: "unbound_cookiesecrets.txt"
```

### Treat DNSSEC alias chains strictly

Expect YXDOMAIN only with DNAME. Revoked DNSKEY signatures are rejected, and
DNAME-to-CNAME plus wildcard-CNAME chains receive stricter trust checks.

## Transport quick reference

### Enable DNS over QUIC

The build needs libngtcp2 and a QUIC-capable OpenSSL. Configure a listener and
memory budget:

```conf
server:
    quic-port: 853
    quic-size: 8m
```

A build without DoQ support ignores QUIC ports and warns when `quic-port` is
set. Confirm operation through `num.query.quic` and `mem.quic`.

### Override transport per forward zone

```conf
server:
    tcp-upstream: no
    tls-upstream: no

forward-zone:
    name: "."
    forward-tcp-upstream: yes
    forward-tls-upstream: yes
```

Per-zone settings override the global upstream transport selection.

### Renew listener certificates with a reload

Reload processing detects certificate-file changes and rebuilds TLS contexts
for DoT, DoH, DoQ, and outgoing DoT. `fast_reload` handles
`tls-service-key`, `tls-service-pem`, and `tls-cert-bundle`.

## Cache and recursion quick reference

### Use secure serve-expired behavior

Expired data can be served with the RFC 8767-oriented defaults. Current
delegation and validation-recursion data can replace expired state, which may
leave less older DNSSEC-validated data available for later fallback.
`serve-expired-reply-ttl` never exceeds the original record TTL.

### Understand ECS cache placement

If an ECS subquery cannot be created, the result is `SERVFAIL`. For
`0.0.0.0/0`, untreated data enters the global cache; data with configured
subnet treatment enters the subnet cache.

### Inspect a cache selectively

```sh
unbound-control cache_lookup example.com
unbound-control cache_lookup +t .
```

The `+t` form accepts TLD and root names, and results include matching subnet
cache data.

## Policy and zone quick reference

Use this module order when RESPIP or RPZ must process DNS64-synthesized
answers:

```conf
server:
    module-config: "respip dns64 validator iterator"
```

Do not assume that inserting cachedb as
`"respip dns64 validator cachedb iterator"` works.

Review these policy details during upgrades:

- Tagged RPZ zones honor their tags.
- RPZ local-CNAME rewrites follow resulting CNAME chains.
- `always_refuse` zones block DS queries too.
- Dynamically added `always_nxdomain` zones locate their parent correctly.
- ZONEMD records in RPZ input are ignored as policy types.
- An ineffective `nodefault` local-zone declaration produces a warning.

## Operations quick reference

### Prefer fast reload for supported changes

```sh
unbound-control fast_reload
```

Configuration is read in a separate thread and service threads pause only
briefly. Dnstap changes propagate to workers, as do supported certificate and
iterator-limit settings.

### Use cache tools without assuming a long global stall

`dump_cache` periodically releases cache locks and separates file-descriptor
activity from lookups. `flush_negative` reports removed data correctly.
Remote-control commands that accept no arguments reject extra arguments.

### Watch the new counters

Depending on enabled features, collect:

- `num.query.quic` and `mem.quic`
- `num.dns_error_reports`
- discard-timeout and wait-limit statistics
- `num.queries.replyaddr_limit`
- `requestlist.current.replies`

## Validation checklist

- Run `unbound-checkconf` and resolve every warning before reload.
- Confirm the effective module order and listener protocol support.
- Exercise one normal, one DNSSEC, and one policy-affected query.
- Verify cache behavior with `cache_lookup` where the task changes TTLs,
  cachedb, ECS, or serve-expired handling.
- Check statistics for QUIC, wait-limit, discard-timeout, mesh, and error-report
  activity as applicable.
- Confirm that a reload actually picked up certificates, dnstap settings, and
  changed limits.
- On packaged systems, inspect service-unit ordering and capabilities rather
  than assuming generated templates replaced local copies.
