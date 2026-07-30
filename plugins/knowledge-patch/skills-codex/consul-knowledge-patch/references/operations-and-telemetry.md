# Operations and Telemetry

## Agent HTTP timeout defaults

Agent `http_config` uses these defaults in 2.0.0:

| Setting | Default |
| --- | --- |
| `read_timeout` | 15 minutes |
| `write_timeout` | 15 minutes |
| `read_header_timeout` | 10 seconds |
| `idle_timeout` | 120 seconds |

The read and write defaults increased from 30 seconds so long-polling blocking
queries are not terminated early. All four settings remain configurable.

## Cluster-wide RPC rate limiting

Enterprise provides a Raft-replicated `rate-limit` configuration entry in
2.0.0. It changes cluster-wide RPC limits at runtime without restarting every
server and can exempt critical RPC methods.

List method names that can be targeted with:

```text
GET /v1/internal/rpc/methods
```

The request requires an ACL token with `operator:read`.

## Manual license-utilization reporting

Enterprise operators can generate and submit license-utilization data through
`/v1/operator/utilization` in 1.22.0. Air-gapped environments can instead
create a census utilization bundle for manual submission:

```shell
consul operator utilization [-today-only] [-message] [-y]
```

Census metrics collection is always enabled. Exporting the collected data for
reporting remains configurable.

## Product telemetry

Self-managed Enterprise clusters can send anonymized product-usage telemetry
in 2.0.0. Reporting is disabled by default and must be explicitly enabled.
Treat the opt-in setting separately from census collection and utilization
export.

## Certificate expiration telemetry

The existing `/agent/metrics` endpoint exposes Prometheus metrics in 2.0.0 for:

- active root CAs
- active signing CAs
- agent TLS certificates
- leaf-certificate renewal health

Metrics carry datacenter, partition, and namespace labels. Certificate
monitoring in the agent `telemetry` block can also emit structured logs with
configurable severity thresholds. The Connect CA API exposes the `NotAfter`
values for root and intermediate certificates.

Use the metrics for alerting, structured logs for thresholded operational
events, and CA API expiration values when inspecting specific certificate
chains.

## IBM PAO licenses

Enterprise licensing and utilization reporting can parse and report IBM
Passport Advantage Online licenses in 2.0.0.
