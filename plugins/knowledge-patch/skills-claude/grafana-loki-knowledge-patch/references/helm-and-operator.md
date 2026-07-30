# Helm and Operator deployment

## Rendering and ownership

- In 3.4.0 chart behavior, ConfigMap and Secret checksums cover only `.data`.
- The installation manager, not the chart template, sets the `managed-by`
  marker.
- Ruler and index-gateway templates include their namespace.
- Headless backend gRPC ports declare `appProtocol: tcp`.
- Ruler configuration renders only when the ruler is enabled and defaults its
  WAL directory.
- `test.enabled=false` suppresses the test pod.
- In 3.5.0, `tpl()` evaluation is restored for read, write, and backend pods,
  and provisioners can be namespaced.
- In 3.6.0, the chart also applies `tpl` to `pattern_ingester`,
  `ingester_client`, and `loki.operational_config`.
- In 3.7.0, `nameOverride` is evaluated with `tpl`.

Render the final manifests whenever a value can contain templates; validation
of values alone will not expose expansion and escaping problems.

## Workload topology and rollout

- The 3.4.0 chart adds overrides-exporter support.
- `topologySpreadConstraints` are available for admin-api pods and distributed
  deployments in 3.4.0, then for SingleBinary in 3.7.0.
- Zone-aware replication splits the ingester HPA in 3.4.0.
- Rollout-group values and ingester names can be prefixed in 3.4.0.
- The canary can run as a Deployment instead of a DaemonSet and batch pushes
  in 3.6.0.
- User namespaces are supported in 3.6.0.
- Configurable init containers cover backend, bloom, distributor, query, read,
  and write workloads in 3.6.0.
- Distributor and read workloads gain startup probes in 3.7.0.
- The canary readiness probe is configurable in 3.7.0.
- Filesystem group change policy defaults to `OnRootMismatch` in 3.7.0.

## Services, DNS, and images

- The query-frontend gRPC load-balancing port can be toggled in 3.7.0.
- Service `trafficDistribution` can be set in 3.7.0.
- `dnsConfig` renders for backend, read, write, SingleBinary, and table-manager
  workloads in 3.7.0.
- The global image registry applies to sidecars in 3.7.0.

## Persistence

- PVC access modes and claim-template labels are configurable in 3.6.0.
- PVCs are retained when a StatefulSet scales down but remain deletable with
  the StatefulSet (3.6.0). Align reclaim policy and backups with that
  asymmetry.
- `volumeAttributesClassName` can be set on volume claim templates in 3.7.0.

## Storage and ruler chart integration

- The chart exposes the full storage configuration in 3.6.0 and can bypass
  generated S3, GCS, and Azure settings.
- Separate ruler storage is supported in 3.6.0.
- Ruler pods can run the rules sidecar, and alert rules can carry custom
  annotations (3.6.0).
- Chunk bucket names are optional in 3.7.0 when using an S3 URL, MinIO, or
  local disk.
- Ruler bucket names are optional in 3.7.0 with local ruler storage.

## Authentication and caching values

- External Memcached and an L2 chunks cache can be configured in 3.6.0.
- Tenant authentication can accept a password hash instead of plaintext in
  3.6.0.

## Loki Operator identity and ingestion

- Managed GCP Workload Identity is supported in 3.4.0.
- The Operator places the log-level attribute in structured metadata in 3.4.0.
- It can drop OTLP attributes, configure a TLS CA for Swift, and enable
  time-based stream sharding in 3.5.0. Attribute dropping is a breaking
  behavior and requires an upgrade review.
- Generated sizing keeps delete workers nonzero and corrects the minimum
  available ingesters for the `1x.pico` size in 3.5.0.

## Loki Operator network, storage, and authorization

- A LokiStack can include Operator-managed NetworkPolicies in 3.6.0.
- S3 secrets can request virtual-host-style access in 3.6.0.
- LokiStack authorization can use OpenTelemetry semantics in 3.6.0.
- The Operator can suppress ingress and customize the gateway server
  certificate in 3.7.0.
- Metrics authentication no longer depends on `kube-rbac-proxy` as of 3.7.3.
- On OCP 4.20, NetworkPolicies are no longer deployed automatically.
- AWS STS deployments receive the region through an environment variable in
  3.7.0.

## OpenShift upgrade review

Default OpenShift stream labels change in 3.7.0 and the change is breaking.
Before reconciliation, compare selectors, alerting rules, dashboards,
retention rules, and cardinality estimates. Separately verify network isolation
on OCP 4.20 because the prior automatic NetworkPolicy behavior no longer
applies.
