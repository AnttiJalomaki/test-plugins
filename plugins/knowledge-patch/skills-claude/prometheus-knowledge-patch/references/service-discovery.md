# Service Discovery

## Kubernetes

### API compatibility

Kubernetes discovery no longer supports `discovery.k8s.io/v1beta1`
EndpointSlices or `networking.k8s.io/v1beta1` Ingresses (since 3.0.0). Clusters
that expose only those beta APIs cannot serve the corresponding discovery
roles.

### Target construction and metadata

Endpoint discovery recognizes sidecar containers when constructing targets
(since 3.0.0).

Discovery can attach namespace metadata (since 3.6.0):

```yaml
kubernetes_sd_configs:
  - role: pod
    attach_metadata:
      namespace: true
```

Pod-role configurations accept node-role selectors (since 3.11.0). Pod targets
also expose `__meta_kubernetes_pod_deployment_name`,
`__meta_kubernetes_pod_cronjob_name`, and
`__meta_kubernetes_pod_job_name` for controller identities.

Kubernetes discovery no longer creates duplicate targets for `*DualStack`
EndpointSlice policies (since 3.11.0).

## AWS

### Unified and provider-specific roles

The unified AWS discovery configuration covers EC2, Lightsail, and ECS (since
3.8.0). AWS discovery adds MSK in 3.10.0 and ElastiCache and RDS roles in
3.11.0. EC2 discovery again honors its configured `endpoint` in 3.11.0.

RDS discovery can filter instances (since 3.13.0), narrowing results before
target processing.

### Addresses and credentials

EC2 discovery can use IPv6 target addresses (since 3.12.0). When both address
families exist, private IPv4 remains the default.

ECS, MSK, RDS, and ElastiCache configurations accept optional `external_id`
(since 3.12.0).

### Custom builds

Use the `remove_all_sd` Go build tag to remove every bundled provider, then
restore selected providers with `enable_<sd name>_sd` tags (since 3.10.0). This
supports smaller custom binaries.

## Azure

Azure service discovery supports Azure Workload Identity (since 3.11.0).
System-assigned managed identity also works with an empty `client_id`. Remote
write has separate Azure identity and certificate options described in the
remote-storage reference.

## Consul

Consul catalog discovery supports server-side catalog filters, reducing results
before relabel processing (since 3.0.0).

From 3.11.2, use `health_filter` for Health API filtering (`3.11.0`). The
general `filter` parameter is no longer incorrectly applied to that API.

## Hetzner

HCloud discovery accepts `label_selector` for server-side filtering (since
3.5.0):

```yaml
hetzner_sd_configs:
  - role: hcloud
    label_selector: environment=production
```

For the `robot` role, migrate `__meta_hetzner_datacenter` to
`__meta_hetzner_robot_datacenter`; the old form remains for compatibility
(`3.11.0`). The HCloud form of `__meta_hetzner_datacenter` was scheduled to stop
working after July 1, 2026. Replace
`__meta_hetzner_hcloud_datacenter_location` and
`__meta_hetzner_hcloud_datacenter_location_network_zone` with
`__meta_hetzner_hcloud_location` and
`__meta_hetzner_hcloud_location_network_zone`.

## Scaleway

Scaleway targets expose `__meta_scaleway_instance_public_ipv4_addresses` and
`__meta_scaleway_instance_public_ipv6_addresses` (since 3.3.0). The singular
`__meta_meta_scaleway_instance_public_ipv4` is no longer set when the public
address is IPv6.

## OpenStack

OpenStack discovery includes Octavia load balancers (since 3.2.0).

## STACKIT

STACKIT Cloud discovery is available from 3.5.0. Fixed 3.12 releases stop
exposing its credentials in plaintext through `/-/config` (`3.12.0`); upgrade
before using it on an endpoint where configuration can be viewed.

## DigitalOcean and Outscale

DigitalOcean Managed Databases can be discovered (since 3.12.0). Outscale Cloud
VM discovery uses `outscale_sd_configs` (since 3.12.0).

## Discovery observability and lifecycle

Most `prometheus_sd_refresh` metrics carry a `config` label with the job name
(since 3.9.0). `prometheus_sd_last_update_timestamp_seconds` records the last
update delivered to consumers (since 3.11.0).

When a scrape job is removed, its per-job `prometheus_sd_refresh*` and
`prometheus_sd_discovered_targets` series are removed as well (since 3.12.0).

The target UI can display the complete relabel trace for a discovered or dropped
target (since 3.8.0).
