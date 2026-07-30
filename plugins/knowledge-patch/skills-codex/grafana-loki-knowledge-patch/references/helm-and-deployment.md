# Helm and Deployment

Render the chart with the exact values used by the installation and inspect the
resulting workloads, services, storage configuration, and Loki configuration.

## Ownership and checksum behavior

Since 3.4.0, ConfigMap and Secret checksums are computed only over `.data`.
Changes outside that field do not affect these rollout checksums.

The installation manager, not a chart template, sets `managed-by`. Do not add
a competing template-owned value when integrating the chart with a manager.

Ruler and index-gateway templates include their namespace. Headless backend
gRPC ports declare `appProtocol: tcp`.

## Ruler and test rendering

Ruler configuration renders only while the ruler is enabled and has a default
WAL directory (3.4.0). Setting `test.enabled=false` suppresses the test pod.

Since 3.6.0, ruler pods can run the rules sidecar, alert rules may carry custom
annotations, and ruler storage can be separate from the main Loki storage.

## Deployment controls

The chart added overrides-exporter support in 3.4.0. It also exposes
`topologySpreadConstraints` for admin-api pods and distributed deployments.
Zone-aware replication splits the ingester HPA, and rollout-group values and
ingester names can use a prefix.

`topologySpreadConstraints` also applies to SingleBinary in 3.7.0. Verify that
topology keys and selectors match the cluster's labels.

## Templated values and namespacing

In 3.5.0, `tpl()` evaluation was restored for read, write, and backend pods,
and provisioners can be namespaced.

In 3.6.0, `tpl` applies to:

- `pattern_ingester`;
- `ingester_client`; and
- `loki.operational_config`.

In 3.7.0, `nameOverride` is evaluated with `tpl`. Treat all of these values as
templates and validate their rendered form, especially when values contain
literal template delimiters.

## Object storage and ruler storage

Use `object_store.storage_prefix` instead of the former `object_store.prefix`
as of 3.5.0.

The chart exposes the full storage configuration in 3.6.0. It can bypass the
generated S3, GCS, and Azure configuration and supports separate ruler
storage. Inspect the effective configuration to ensure the bypass path does
not coexist with unintended generated values.

Chunk bucket names are not required in 3.7.0 when using an S3 URL, MinIO, or
local disk. Ruler bucket names are optional with local ruler storage. The
global image registry also applies to sidecars.

## Services and monitoring

The nginx Service no longer receives a ServiceMonitor as of 3.5.0. Add
monitoring explicitly if the installation still requires nginx service
metrics.

In 3.7.0, the chart can toggle the query-frontend gRPC load-balancing port and
set the Service `trafficDistribution` field. Ensure clients and traffic policy
match the rendered ports.

## Caches and authentication

The chart supports external Memcached and an L2 chunks cache as of 3.6.0.
Validate cache addresses, fallback behavior, and consistency expectations for
both cache levels.

Tenant authentication may receive a password hash instead of a plaintext
password. Supply the form expected by the selected chart values and gateway
configuration.

## Block builder and Operational UI

The chart exposes `block_builder` configuration for deploying Kafka-backed
record consumption in 3.6.0.

Enabling the Operational UI in the chart enables its server APIs on queriers;
the gateway forwards UI requests to them. The UI JavaScript itself is provided
by a Grafana plugin.

## Canary controls

The canary can run as a Deployment instead of a DaemonSet and can batch log
pushes as of 3.6.0.

In 3.7.0, its `readinessProbe` is configurable, and the canary accepts an
arbitrary label set for its query. Keep the pushed and queried labels aligned
when customizing either side.

## Init containers and user namespaces

User namespaces are supported as of 3.6.0. Configurable init containers are
available across backend, bloom, distributor, query, read, and write
workloads. Check volume mounts, security contexts, and startup dependencies in
each workload that receives an init container.

## Probes and pod startup

Distributor and read workloads gained startup probes in 3.7.0. The filesystem
group change policy is `OnRootMismatch`; account for existing volume ownership
rather than expecting a recursive ownership change on every start.

## Persistence

PVC access modes and claim-template labels are configurable as of 3.6.0. PVCs
are retained when a StatefulSet scales down but remain deletable with the
StatefulSet.

In 3.7.0, `volumeAttributesClassName` can be set on volume claim templates.
Confirm storage-driver support before applying it.

## DNS configuration

`dnsConfig` renders for backend, read, write, SingleBinary, and table-manager
workloads as of 3.7.0. Validate every affected pod template when introducing
custom resolvers or search domains.

## Memberlist and IPv6

Memberlist respects configured interface names when selecting its advertise
address as of 3.4.0. In 3.5.0, IPv6 interfaces listed in
`common.instance_interface_names` are valid advertise-address sources, and the
query frontend can resolve IPv6 addresses.

`common.instance_enable_ipv6` is propagated to every component as of 3.6.0.
Review the rendered setting across mixed component types.

## Chart repository and meta-monitoring

Effective March 16, 2026, the open-source Loki chart moved to
`grafana-community/helm-charts` for community maintenance. Update repository
references and automation; the GEL chart remains separately maintained.

Meta-monitoring responsibilities moved from the Grafana meta-monitoring chart
to the Grafana Kubernetes Monitoring chart in 3.6.0. Migrate values and release
ownership rather than continuing to extend the former chart.
