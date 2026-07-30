# Zones and resolution

## Subnet cache module

Since 1.23.0, `module-config` defaults to `"validator iterator"` regardless
of `--enable-subnet`. A build that needs the subnet module must name it
explicitly:

```conf
server:
    module-config: "subnetcache validator iterator"
```

## Default local zones

Since 1.23.0, `resolver.arpa` and `service.arpa` are served locally by
default. An upgrade can therefore change whether those names are forwarded.

## Forwarding and stub zones

### Auth-zone-aware forwarding

Since 1.23.0, forwarder checks take configured auth-zones into account.

### Hostname dependency warnings

Since 1.24.0, Unbound detects and warns when a stub or forward zone names its
servers with hostnames in a way that can create a circular resolution
dependency.

## Auth-zones

### Status and acquisition time

Since 1.24.0, status reporting is available for auth-zones. The acquired
timestamp is set only after the zonefile has been read.

### Empty-label origin in downloaded files

Since 1.24.0, auth-zone files downloaded over HTTP may use an empty-label
`$ORIGIN`.

### Hostnames in `allow-notify`

Since 1.25.0, hostname-form `allow-notify` entries retain both resolved IPv4
and IPv6 addresses. They are also looked up when an auth-zone has only URL
transfer sources.

## DNS64 and NAT64

### Response policy module order

Since 1.24.0, RESPIP and RPZ can process DNS64-synthesized answers with:

```conf
server:
    module-config: "respip dns64 validator iterator"
```

Using `"respip dns64 validator cachedb iterator"` is explicitly not known to
work.

### Retry exclusions

Since 1.25.0, retries consistently apply the interaction between `do-nat64`
and `do-not-query-address`.

## RESINFO

Since 1.23.0, Unbound supports RESINFO RR type 261. The
`LDNS_RR_TYPE_RESINFO` representation is TXT-like.
