# Helm, operations, and security

## Upgrade compatibility

### Support window

Only the newest minor release is supported; the next minor automatically
deprecates its predecessor. At the `operations-and-security` snapshot, 2.8 is
the supported line, guarantees Kubernetes 1.35, and reaches end of life when
2.9 ships. Image rebuilds, Go dependency updates, and security and bug fixes
apply to that supported line. Upgrade one minor at a time.

### Deprecation scope

The protected surface consists of API specs, status and conditions, enums and
constants, controller flags and environment variables, metrics, and documented
`ExternalSecret` update behavior. Helm charts, releases, images, signatures,
OLM builds, source imports, and unspecified behavior are outside that
guarantee (`operations-and-security`).

Introducing a deprecation requires a minor release during 0.x or a major
release from 1.x onward. Only in-scope removals inherit Kubernetes deprecation
timelines. ESO remains classified as beta: features are enabled by default and
considered safe to enable, but schemas or semantics can change incompatibly
with migration instructions, and the policy does not recommend beta software
for production.

### Removed and relocated artifacts

- Alibaba and Device42 providers were removed in 2.0.0.
- The default image repository moved in 1.1.0 from
  `oci.external-secrets.io/external-secrets/external-secrets` to
  `ghcr.io/external-secrets/external-secrets`. The Helm repository remains on
  GitHub Pages; migrate pinned image overrides because the old domain is only
  temporarily available.

### Flux OCIRepository

Flux consumers upgrading to 2.2.0 must select and extract the Helm chart
content layer:

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

## CRDs, reconcilers, and admission

CRD creation defaults on, but disabling a CRD does not disable its reconciler.
Pair each disabled `crds.create*` setting with its corresponding `process*`
setting or the controller logs errors about the missing CRD. Disabling the
webhook also requires disabling CRD conversion, or the API server continues
calling a conversion endpoint that does not exist
(`operations-and-security`).

```yaml
crds:
  createPushSecret: false
  conversion:
    enabled: false
processPushSecret: false
webhook:
  create: false
```

- `processClusterGenerator` controls cluster-scoped generator reconciliation
  (since 0.20.0).
- `processClusterExternalSecret` also gates chart write RBAC for
  `externalsecrets` (since 2.5.0).
- The chart can control installation behavior when Prometheus CRDs are missing
  (since 0.20.0).
- The validating webhook obtains the `SecretStore` failure policy dynamically
  (since 2.0.0), and the chart applies `failurePolicy` to the
  `ClusterSecretStore` webhook (since 2.4.0).
- `ValidatingWebhookConfiguration` supports annotations (since 0.16.0).

## Chart workload customization

- Configure controller init containers through chart values (since 0.19.0).
- Configure `hostUsers` and certificate algorithms (since 1.3.0).
- Configure pod `hostAliases` (since 2.0.0).
- Global values are available for shared deployment configuration (since
  1.2.0).
- `storeRequeueInterval` controls store requeue cadence (since 2.7.0).
- Grafana dashboard resources accept extra labels for installations with
  distinct selectors (since 0.20.0).
- The chart can control RBAC aggregation into the Kubernetes admin role with
  `aggregateToAdmin` (since 2.8.0).

## Probes and disruption budgets

- The controller has a liveness probe (since 0.20.0).
- The chart configures a readiness probe for the controller Deployment (since
  2.2.0).
- Cert-controller and webhook liveness probes are available (since 2.4.0).
- The webhook Deployment supports a configurable startup probe (since 2.8.0).
- PodDisruptionBudget values render correctly as percentages (since 0.18.0).
- Explicit zero values for `minAvailable` or `maxUnavailable` still render the
  PDB spec (since 2.6.0).

The controller defaults to one replica with leader election, liveness,
readiness, and PDB disabled. Webhook and cert-controller readiness default on,
but both default to one replica with liveness and PDB disabled. Enable
availability controls deliberately (`operations-and-security`):

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

The controller exposes `--leader-election-id` for independent HA installations
(since 2.4.0). Leader-election lease timings are configurable (since 2.8.0).
The chart also has a cert-manager leader flag (since 2.1.0). Deployments sharing
a namespace need distinct lease IDs.

## Namespace-scoped installation and RBAC

Namespaced resources cannot make cross-namespace references to namespaced
stores, Secrets, or referents. Cluster-scoped resources need separate review
because they can span namespaces. A namespace-only installation creates scoped
roles and implicitly disables cluster-scoped controllers
(`operations-and-security`):

```yaml
scopedRBAC: true
scopedNamespace: payments
```

If `scopedRBAC` is enabled and `scopedNamespace` is unset, the chart defaults
it to `.Release.Namespace` (since 2.5.0).

Cert-controller RBAC is limited to its managed CRDs and the webhook Secret
(since 2.7.0). The create rule for `serviceaccounts/token` renders
conditionally (since 2.5.0).

### Delegate TokenRequest narrowly

Authentication through `serviceAccountRef` needs TokenRequest access. The
default role can create tokens for any ServiceAccount in its scope. Disable
that blanket rule and grant token creation only for each referenced account
(`operations-and-security`):

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
```

## Pod and controller authority

Default pod security contexts use the restricted profile: UID 1000, non-root,
read-only root filesystem, no privilege escalation, all capabilities dropped,
and `RuntimeDefault` seccomp. This does not make the chart a hardened
deployment: NetworkPolicies and metrics TLS/authentication default off, while
broad ServiceAccount token creation and aggregation into view, edit, and admin
roles default on (`operations-and-security`).

`genericTargets.enabled` defaults false. Enabling it grants create, update, and
delete access to ConfigMaps plus configured verbs for every resource in
`genericTargets.resources`. Treat each target API group as an explicit
privilege expansion and apply suitable encryption and admission controls
(`operations-and-security`).

## Network and serving security

The controller needs outbound access to the Kubernetes API and selected
providers; webhook and cert-controller need the API. Prefer private provider
endpoints and allow DNS plus only required API/provider destinations. Expected
inbound ports are:

- controller metrics 8080 and optional health 8082;
- webhook admission 10250, metrics 8080, and health 8081;
- cert-controller metrics 8080 and health 8081.

Policy engines should deny unused providers, constrain remote-key prefixes,
and restrict `ClusterSecretStore` use (`operations-and-security`).

- Secure metrics serving is supported (since 0.20.0).
- Metrics authentication and authorization can use `FilterProvider` (since
  2.5.0).
- HTTP/2 serving is configurable and can be disabled for the deployment's
  security posture (since 0.20.0).
- The chart can create an optional NetworkPolicy (since 2.8.0).

## Metrics and service metadata

- Cert-controller metrics Service annotations render correctly (since 2.1.0).
- Keeper exports `provider_api_calls_count` for provider API-call volume (since
  2.6.0).

## Provider maturity and capability checks

At the 2.8 `operations-and-security` snapshot, AWS Secrets Manager, AWS
Parameter Store, Akeyless, Azure Key Vault, CyberArk Secrets Manager, GCP
Secret Manager, HashiCorp Vault, IBM Cloud Secrets Manager, Oracle Vault, and
Previder are stable. Kubernetes and SecretServer are beta; every other listed
provider is alpha.

Maturity is not feature parity. Check find, metadata fetch, referent
authentication, store validation, push, merge, and delete support
independently.

## Release artifact verification

Images carry keyless Cosign signatures, SLSA provenance attestations, and SPDX
JSON SBOM attestations. Verify an immutable digest, certificate issuer
`https://token.actions.githubusercontent.com`, and a subject identifying the
External Secrets release workflow on `refs/heads/main`
(`operations-and-security`). Signatures and provenance themselves are outside
the deprecation guarantee.
