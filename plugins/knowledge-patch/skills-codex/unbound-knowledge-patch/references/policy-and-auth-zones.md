# Policy and auth zones

## RPZ and response-IP

Tags assigned to tagged RPZ zones are honored (since 1.21.0). This corrects a
regression in which tags were ignored after moving from 1.19.3 to 1.20.0.

When an RPZ local-CNAME rewrite introduces an alias, Unbound follows the CNAME
chain rather than ending resolution at the rewritten CNAME (since 1.24.0).

RESPIP and RPZ policy applies to DNS64-synthesized answers when the processing
order is (since 1.24.0):

```conf
server:
    module-config: "respip dns64 validator iterator"
```

The order `"respip dns64 validator cachedb iterator"` is explicitly not known
to work.

ZONEMD is ignored as a policy type when RPZ zones are loaded, preventing a
ZONEMD-bearing policy zone from disrupting root-key priming (since 1.25.0).

## Local zones

An `always_nxdomain` zone added through `unbound-control` finds its parent
correctly, making dynamic blocking reliable (since 1.21.0).

`resolver.arpa` and `service.arpa` are served as local zones by default (since
1.23.0). This can change whether queries for those names reach a forwarder.

An `always_refuse` local zone blocks DS queries in addition to other query
types (since 1.25.0).

`unbound-checkconf` warns when a `nodefault` local-zone declaration has no
effect (since 1.25.0).

## Auth zones and forwarding

Forwarder checks take configured auth zones into account (since 1.23.0).

Auth-zone status is available for operational inspection, and the acquired
timestamp is assigned only after the zonefile is read (since 1.24.0).

An auth-zone file downloaded over HTTP may use an empty-label `$ORIGIN` (since
1.24.0).

Hostname values in `allow-notify` retain their resolved IPv4 and IPv6
addresses (since 1.25.0). The hostnames are resolved even when an auth zone
uses only URL-based transfer sources.

## Rebinding protection

`private-address` filtering removes matching SVCB and HTTPS records as well as
address records (since 1.25.0). This closes the rebinding path that used
service-binding records to avoid address-record filtering.

## PF and ipset policy integration

The ipset integration supports BSD PF tables for response-IP and ipset policy
workflows (since 1.21.0).
