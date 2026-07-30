# Service discovery

Use this reference when configuring provider roles, relabeling discovery
metadata, monitoring refresh behavior, or building a Prometheus binary with a
reduced provider set.

## Kubernetes

### API and target compatibility

Prometheus no longer supports `discovery.k8s.io/v1beta1` EndpointSlices or
`networking.k8s.io/v1beta1` Ingresses from 3.0.0. Clusters exposing only those
beta APIs are incompatible with the corresponding discovery roles.

Endpoint discovery recognizes sidecar containers when constructing discovered
targets from 3.0.0.

Kubernetes DualStack EndpointSlice policies no longer create duplicate targets
from 3.11.0.

### Metadata and selectors

Kubernetes discovery can attach namespace metadata from 3.6.0:

```yaml
kubernetes_sd_configs:
  - role: pod
    attach_metadata:
      namespace: true
```

Pod-role configurations accept node-role selectors from 3.11.0.

Pod targets expose these controller labels from 3.11.0:

- `__meta_kubernetes_pod_deployment_name`
- `__meta_kubernetes_pod_cronjob_name`
- `__meta_kubernetes_pod_job_name`

## AWS

### Unified discovery and roles

Prometheus 3.8.0 introduces a unified AWS service-discovery option for EC2,
Lightsail, and ECS.

AWS discovery adds these provider roles:

- MSK in 3.10.0
- ElastiCache and RDS in 3.11.0

EC2 discovery again honors its configured `endpoint` from 3.11.0.

RDS discovery can filter instances from 3.13.0.

### Identity and addressing

ECS, MSK, RDS, and ElastiCache discovery configurations accept optional
`external_id` from 3.12.0.

EC2 discovery supports IPv6 target addresses from 3.12.0. When both IP
families are available, private IPv4 remains the default.

## Azure

Azure service discovery supports Azure Workload Identity from 3.11.0.
System-assigned managed identity also works with an empty `client_id`.

## Consul

Consul catalog discovery supports server-side catalog filters from 3.0.0,
allowing the server to narrow results before target processing.

From 3.11.2, use `health_filter` for Consul Health API filtering. The general
`filter` parameter is no longer incorrectly applied to that API.

## Hetzner

HCloud discovery accepts `label_selector` from 3.5.0:

```yaml
hetzner_sd_configs:
  - role: hcloud
    label_selector: environment=production
```

The 3.11.0 metadata migration changes role-specific labels:

- For `robot`, replace `__meta_hetzner_datacenter` with
  `__meta_hetzner_robot_datacenter`; the old label remains as a compatibility
  alias.
- The HCloud form of `__meta_hetzner_datacenter` was scheduled to stop working
  after July 1, 2026.
- Replace `__meta_hetzner_hcloud_datacenter_location` with
  `__meta_hetzner_hcloud_location`.
- Replace `__meta_hetzner_hcloud_datacenter_location_network_zone` with
  `__meta_hetzner_hcloud_location_network_zone`.

Update relabeling rules before depending on the new names.

## STACKIT

STACKIT Cloud discovery is available from 3.5.0.

Affected builds expose STACKIT discovery credentials in plaintext through
`/-/config`; STACKIT users need a fixed 3.12.0 release.

## OpenStack, Scaleway, DigitalOcean, and Outscale

OpenStack discovery supports Octavia load balancers from 3.2.0.

Scaleway discovery exposes
`__meta_scaleway_instance_public_ipv4_addresses` and
`__meta_scaleway_instance_public_ipv6_addresses` from 3.3.0. It no longer sets
`__meta_meta_scaleway_instance_public_ipv4` when the public address is IPv6.

DigitalOcean Managed Databases discovery is available from 3.12.0.

Outscale Cloud VM targets can be discovered with `outscale_sd_configs` from
3.12.0.

## Relabeling diagnostics and refresh metrics

The target UI can display every relabeling step for a discovered target from
3.8.0, showing how labels changed and why a target was dropped.

Most `prometheus_sd_refresh` metrics gain a `config` label containing the job
name from 3.9.0, allowing refresh behavior to be attributed to one discovery
configuration.

Use `prometheus_sd_last_update_timestamp_seconds` from 3.11.0 to monitor the
last time a discovery update was sent to consumers.

From 3.12.0, per-job `prometheus_sd_refresh*` and
`prometheus_sd_discovered_targets` series are removed when their scrape job is
deleted.

## Custom builds

From 3.10.0, a custom build can exclude all bundled discovery providers with
the `remove_all_sd` Go build tag, then restore selected providers with
`enable_<sd name>_sd` tags. This reduces binary size when only a known provider
set is needed.

## Redirect behavior

Discovery HTTP clients strip credentials and configured headers when a redirect
changes host from 3.13.0. Provider integrations must not rely on cross-host
credential forwarding.
