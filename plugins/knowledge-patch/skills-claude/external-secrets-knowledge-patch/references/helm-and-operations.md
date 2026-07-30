# Helm and Operations

Use this reference when installing, upgrading, scoping, or operating ESO. Render
the chart and inspect live CRDs: chart values, controller processes, admission,
and conversion are related but independently configurable.

## Image and chart distribution

### Image repository migration

The chart's default controller image moved from
`oci.external-secrets.io/external-secrets/external-secrets` to
`ghcr.io/external-secrets/external-secrets` (1.1.0). The old image domain is
being retired and is only temporarily available. Update repository overrides
and mirrors; the Helm chart repository remains on GitHub Pages.

### Flux OCIRepository

Flux installations that fetch the chart through `OCIRepository` must select and
extract the Helm chart content layer (2.2.0):

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: eso-oci
spec:
  interval: 1m0s
  provider: generic
  ref:
    tag: 2.2.0
  url: oci://ghcr.io/external-secrets/charts/external-secrets
  layerSelector:
    mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip
    operation: extract
```

## CRDs, controllers, webhooks, and conversion

CRD installation defaults on. Disabling a CRD does not disable its reconciler;
pair every disabled `crds.create*` value with the corresponding `process*`
value. Otherwise the controller logs errors because the CRD is absent.
Disabling the webhook also requires disabling CRD conversion, or the API server
continues to call a conversion endpoint that does not exist
(operations-and-security).

```yaml
crds:
  createPushSecret: false
  conversion:
    enabled: false
processPushSecret: false
webhook:
  create: false
```

Related controls:

- `processClusterGenerator` determines whether cluster-scoped generators are
  processed (0.20.0).
- A controller flag enables or disables SecretStore reconciliation (1.2.0).
- The chart handles missing Prometheus CRDs according to a configurable install
  policy (0.20.0); choose it explicitly in clusters without those CRDs.
- `ValidatingWebhookConfiguration` accepts annotations (0.16.0).
- SecretStore webhook `failurePolicy` is obtained dynamically (2.0.0), and the
  chart applies the configured policy to `ClusterSecretStore` admission
  (2.4.0).
- Legacy beta API serving is configurable for the migration window (1.3.0).

## Namespace-scoped installation

Namespaced `ExternalSecret` and `SecretStore` resources cannot make
cross-namespace references to a store, Secret, or other namespaced referent.
Cluster-scoped resources need separate scrutiny because they span namespaces.

For namespace-only operation (operations-and-security):

```yaml
scopedRBAC: true
scopedNamespace: payments
```

Scoped RBAC implicitly disables cluster-scoped controllers. If `scopedRBAC` is
enabled and `scopedNamespace` is absent, the chart defaults the namespace to
`.Release.Namespace` (2.5.0).

## Workload customization

The chart supports:

- init containers for External Secrets deployments (0.19.0);
- global values for shared deployment configuration (1.2.0);
- `hostUsers` control (1.3.0);
- certificate algorithm selection (1.3.0);
- pod `hostAliases` for static hostname mappings (2.0.0).

## Probes, replicas, and disruption budgets

Availability settings are mostly opt-in (operations-and-security):

- The controller defaults to one replica; leader election, liveness, readiness,
  and PodDisruptionBudget are disabled.
- Webhook and cert-controller readiness is enabled, but each defaults to one
  replica with liveness and PDB disabled.

A deliberate controller configuration can look like:

```yaml
replicaCount: 2
leaderElect: true
leaderElectionID: payments-external-secrets
livenessProbe:
  enabled: true
readinessProbe:
  enabled: true
podDisruptionBudget:
  enabled: true
```

Feature details:

- A controller liveness probe is available (0.20.0).
- The chart accepts percentage PodDisruptionBudget values (0.18.0).
- A controller Deployment readiness probe is configurable (2.2.0).
- Webhook and cert-controller liveness probes are available (2.4.0).
- An explicit zero `minAvailable` or `maxUnavailable` still renders the PDB spec
  (2.6.0).
- A webhook Deployment startup probe is configurable (2.8.0).

## Leader election

- A cert-manager leader flag is available (2.1.0).
- `--leader-election-id` allows independent HA deployments to use distinct
  election identities (2.4.0).
- Leader-election lease timings are configurable (2.8.0).

Give multiple ESO installations in the same namespace distinct lease IDs.
Change timing values only after accounting for replica count, failure-detection
time, API latency, and disruption budgets.

## RBAC

### ServiceAccount TokenRequest

The chart renders the `serviceaccounts/token` create rule only when required
(2.5.0). Provider authentication through `serviceAccountRef` needs TokenRequest
access, but the default controller role can create tokens for every
ServiceAccount in its scope. Disable the broad grant and add one named grant per
referenced account (operations-and-security):

```yaml
rbac:
  serviceAccountTokenCreate: false
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: eso-token-provider-reader
  namespace: payments
rules:
  - apiGroups: [""]
    resources: ["serviceaccounts/token"]
    resourceNames: ["provider-reader"]
    verbs: ["create"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: eso-token-provider-reader
  namespace: payments
subjects:
  - kind: ServiceAccount
    name: external-secrets
    namespace: external-secrets
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: eso-token-provider-reader
```

TokenRequest bodies use the URL namespace, fixing namespace inconsistency
(2.6.0). Keep the Role in the target namespace and verify the controller subject
and bound `resourceNames`.

### Controller-specific and aggregate permissions

- Write permissions for `externalsecrets` are gated by
  `processClusterExternalSecret` (2.5.0).
- Cert-controller RBAC is limited to the CRDs it manages and the webhook Secret
  (2.7.0).
- `aggregateToAdmin` controls chart RBAC aggregation into the Kubernetes admin
  role (2.8.0).

The chart still aggregates some access into standard roles by default; review
view, edit, and admin role impact as part of a least-privilege installation.

## Metrics, health, and logging

- Metrics can be served securely (0.20.0).
- Authentication and authorization for metrics can use `FilterProvider`
  (2.5.0).
- Cert-controller metrics Service annotations are applied correctly (2.1.0).
- Target Secret deletion and data-key changes are logged (2.7.0).
- `storeRequeueInterval` exposes store requeue cadence as a chart value (2.7.0).
- HTTP/2 serving is configurable and can be disabled for a security posture
  that forbids it (0.20.0).

Grafana dashboard resources accept extra labels for installations where multiple
Grafana selectors coexist (0.20.0).

## NetworkPolicy and ports

An optional chart NetworkPolicy is available (2.8.0), but it must be adapted to
the actual exfiltration paths. The controller needs egress to the Kubernetes API
and selected secret providers; webhook and cert controller need the API. Allow
DNS and prefer private provider endpoints.

Expected ingress endpoints (operations-and-security):

| Component | Ports |
| --- | --- |
| Controller | metrics `8080`; optional health `8082` |
| Webhook | admission `10250`; metrics `8080`; health `8081` |
| Cert controller | metrics `8080`; health `8081` |

Policy enforcement should also deny unused providers, constrain allowed remote
key prefixes, and restrict `ClusterSecretStore` references.
