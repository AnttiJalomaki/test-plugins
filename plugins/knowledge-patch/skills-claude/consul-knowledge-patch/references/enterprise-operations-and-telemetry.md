# Enterprise Operations and Telemetry

## Change Cluster-Wide RPC Limits at Runtime

Enterprise provides a Raft-replicated `rate-limit` configuration entry
(since 2.0.0). Use it to change cluster-wide RPC rate limits at runtime without
restarting each server. Add exemptions for RPC methods that are critical to
cluster operation.

List the exact method names that can be targeted with:

```text
GET /v1/internal/rpc/methods
```

The request requires an ACL token granting `operator:read`.

## Generate Utilization Reports Manually

Enterprise operators can generate and submit license-utilization data through
`/v1/operator/utilization` (since 1.22.0). For an air-gapped environment,
create a census utilization bundle and submit it manually.

```shell
consul operator utilization [-today-only] [-message] [-y]
```

Census metrics collection is always enabled. Export for reporting remains
configurable, so do not represent disabled export as disabled collection.

## Configure Product-Usage Telemetry

Self-managed Enterprise clusters can send anonymized product-usage telemetry
(since 2.0.0). Reporting is disabled by default and requires explicit opt-in.
Preserve that default unless the operator intentionally enables reporting.

## Monitor Certificate Expiration

The existing `/agent/metrics` endpoint exposes Prometheus metrics for these
certificate states (since 2.0.0):

- Active root and signing certificate authorities
- Agent TLS certificate expiration
- Leaf-certificate renewal health

Metrics include datacenter, partition, and namespace labels. Use these labels
to scope alerts correctly.

Certificate monitoring in the agent `telemetry` block can also emit structured
logs with configurable severity thresholds. The Connect CA API exposes
`NotAfter` values for root and intermediate certificates. Combine metrics,
logs, and API inspection when diagnosing expiration risk.

## Parse IBM PAO Licenses

Enterprise licensing and utilization reporting can parse and report IBM
Passport Advantage Online licenses (since 2.0.0). Preserve the IBM PAO license
form when building reporting or validation workflows.

## Interpret Extended Support Options

The 2.0.0 major version introduces Enterprise support options for longer
contract periods. Earlier Enterprise releases remain maintained under their
existing long-term-support contracts; do not reinterpret those contracts
solely because the product version crossed the new major boundary.
