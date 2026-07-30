# Remote read, remote write, and authentication

Use this reference for remote protocol compatibility, queue and transport
behavior, self-monitoring metric migrations, authentication, and remote-data
validation.

## Transport and storage contracts

### HTTP/2 default

Remote-write HTTP clients default `http_config.enable_http2` to `false` in the
`3.0-migration`. This lets parallel queues use multiple sockets. Set it
explicitly to `true` only when the endpoint needs the v2 transport behavior.

### Remote-read selector contract

TSDB-compatible storage must return only results that match the requested
selectors (`3.0-migration`). Although Prometheus does not explicitly enforce
this contract, a third-party `remote_read` implementation that returns
nonmatching series can cause undefined behavior.

Histograms received through remote read are validated from 3.9.0, so invalid
histogram data is no longer silently accepted.

On the 3.11.0 line, deploy at least 3.11.3 to enforce the declared-length limit
for Snappy remote-read requests (CVE-2026-42154).

### DNS address selection

Remote-write clients can opt into a resolver that randomly selects one returned
IP from 3.1.0. Use it for multi-address destinations that should not always
receive traffic at the first DNS result.

## Remote Write 2 and histogram transport

### Protocol schema and timestamps

The receiver follows the Remote Write 2.0-rc.4 specification from 3.8.0, which
renames created timestamps to start timestamps. Update senders and receiver
integrations to the revised terminology and schema.

From 3.11.0, too-old samples in Remote Write 2 histogram paths return HTTP 400
instead of 500. This tells clients not to retry permanently rejected samples.

### Native histograms

Federation transports custom-bucket native histograms from 3.7.0. Remote Write
1.0 does not support them; Prometheus suppresses those sends and logs a
warning.

Unsupported histogram schema behavior and resolution reduction are detailed in
[Histograms, TSDB, and storage](histograms-tsdb-and-storage.md).

### Type and unit labels

With `--enable-feature=type-and-unit-labels`, outgoing Remote Write 2.0 series
include type and unit labels from 3.7.0. If metadata WAL records conflict with
reserved `__type__` or `__unit__` labels already carried by a series, the
reserved metadata values win (`feature-flags`).

## Queue validation and timing

Prometheus validates `remote_write.queue_config` fields while loading
configuration from 3.12.0. Invalid values are rejected before they can cause a
runtime panic or silent misconfiguration.

`prometheus_remote_storage_sent_batch_duration_seconds` is measured after the
request has been sent from 3.11.0 rather than before it.

## Metric migration

Several remote-write metrics are deprecated from 3.7.0. Update dashboards and
alerts as follows:

- Replace `prometheus_remote_storage_samples_in_total` with
  `prometheus_wal_watcher_records_read_total{type="samples"}` and
  `prometheus_remote_storage_samples_dropped_total`.
- Replace `prometheus_remote_storage_histograms_in_total` with
  `prometheus_wal_watcher_records_read_total{type=~".*histogram_samples"}` and
  `prometheus_remote_storage_histograms_dropped_total`.
- Replace `prometheus_remote_storage_exemplars_in_total` with
  `prometheus_wal_watcher_records_read_total{type="exemplars"}` and
  `prometheus_remote_storage_exemplars_dropped_total`.
- Replace `prometheus_remote_storage_highest_timestamp_in_seconds` with
  `prometheus_remote_storage_queue_highest_timestamp_seconds`. The new metric
  accounts for relabeling and is more accurate.

## Azure authentication

### Managed and workload identities

Remote-write Azure AD authentication accepts an empty `client_id` for a
system-assigned managed identity from 3.5.0:

```yaml
remote_write:
  - url: https://example.invalid/api/v1/write
    azuread:
      managed_identity:
        client_id: ""
```

Azure Workload Identity is supported as a remote-write authentication method
from 3.7.0.

From 3.9.0, AzureAD remote-write authentication can request a custom scope.

Remote write can authenticate to an Azure Monitor Workspace with a certificate
from 3.13.0.

### Secret exposure

On the 3.11.0 line, use at least 3.11.3 to prevent an AzureAD OAuth
`client_secret` from appearing through `/-/config`
(CVE-2026-42151).

## OAuth2 and SigV4

OAuth2 supports the JWT bearer grant defined by RFC 7523 section 3.1 from
3.8.0.

SigV4 configuration supports `use_fips_sts_endpoint` from 3.8.0:

```yaml
sigv4:
  use_fips_sts_endpoint: true
```

Use it when AWS authentication requires a FIPS-compliant STS endpoint.

HTTP-client SigV4 authentication accepts an AWS `external_id` from 3.11.0.
Service-discovery-specific external IDs are covered separately in
[Service discovery](service-discovery.md).

## Redirect credential safety

From 3.13.0, HTTP clients do not forward authorization headers, basic or bearer
credentials, OAuth2 credentials, or configured headers when a redirect changes
host. This applies to remote read and write as well as scraping, alerting, and
service discovery. Integrations must authenticate the destination directly
rather than relying on cross-host credential forwarding.

## Promtool transport

`promtool push metrics` can send Remote Write 2.0 messages from 3.8.0 by
selecting the message type with `--protobuf_message`.
