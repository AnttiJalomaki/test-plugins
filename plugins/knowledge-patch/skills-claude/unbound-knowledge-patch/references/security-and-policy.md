# Security and policy

## EDNS COOKIE controls

### Persistent secret rollover

Since 1.21.0, `cookie-secret-file` persists EDNS COOKIE secrets for rollover:

```conf
server:
    cookie-secret-file: "unbound_cookiesecrets.txt"
```

Rotate at runtime with the `add_cookie_secret`, `activate_cookie_secret`, and
`drop_cookie_secret` remote-control commands. Use `print_cookie_secrets` to
inspect the values currently in use.

### COOKIE-aware IP rate limiting

Since 1.21.0, `ip-ratelimit-cookie` is applied as configured. An existing
configuration that already sets it begins enforcing the intended rate limit
for COOKIE clients after upgrade.

## DNSSEC validation and keys

### Root trust anchor

The `unbound-anchor` compiled-in root keys include key 38696 since 1.21.0.
Inspect the compiled content with:

```sh
unbound-anchor -l
```

### Hardened missing-AAAA glue lookup

Since 1.22.0, `harden-unverified-glue` also governs missing AAAA lookups
started by cache filling. Enabling it therefore hardens this path as well as
the previously covered glue paths.

### RRSIG scrub limit

Since 1.25.0, `iter-scrub-rrsig` caps how many RRSIG records the iterator
scrubber retains. The default is 8:

```conf
server:
    iter-scrub-rrsig: 8
```

### Alias-chain validation

Since 1.25.0:

- YXDOMAIN is accepted only when a DNAME is present.
- Signatures made by revoked DNSKEYs are rejected.
- DNAME-to-CNAME and wildcard-CNAME chains receive stricter trust checks.

## Response policy zones

### Tagged policy matching

Since 1.21.0, tags on tagged RPZ zones are honored again. The regression
caused tags to be ignored after an upgrade from 1.19.3 to 1.20.0.

### RPZ local-CNAME chains

Since 1.24.0, resolution follows CNAME chains introduced by an RPZ
local-CNAME rewrite. It no longer stops at the rewritten CNAME.

### DNS64-synthesized answers

Since 1.24.0, RESPIP and RPZ apply to DNS64-synthesized answers with:

```conf
server:
    module-config: "respip dns64 validator iterator"
```

The cachedb insertion
`"respip dns64 validator cachedb iterator"` is explicitly not known to work.

### RPZ input containing ZONEMD

Since 1.25.0, ZONEMD is ignored as a policy type while loading RPZ zones.
This prevents such input from breaking root-key priming.

## Local-zone policy

### Dynamic `always_nxdomain`

Since 1.21.0, an `always_nxdomain` zone added with `unbound-control` locates
its parent correctly, restoring reliable blocking for dynamically added
zones.

### DS in `always_refuse`

Since 1.25.0, an `always_refuse` local zone blocks DS queries as well as other
query types.

### Ineffective `nodefault`

Since 1.25.0, `unbound-checkconf` warns when a `nodefault` local-zone
declaration has no effect.

## Rebinding protection

Since 1.25.0, `private-address` filtering removes SVCB and HTTPS records that
match configured private-address ranges. This closes a rebinding path that
does not depend on A or AAAA records.

## Contributed cryptography

RFC 9558 ECC-GOST12 support is available since 1.25.0 as
`contrib/gost12.patch`. It replaces the older GOST integration for
deployments that apply the contributed patch.
