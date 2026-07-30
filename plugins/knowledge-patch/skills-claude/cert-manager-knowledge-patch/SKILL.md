---
name: cert-manager-knowledge-patch
description: cert-manager
version: 1.21
license: MIT
metadata:
  author: Nevaberry
---


# cert-manager Knowledge Patch

Use this skill when upgrading, configuring, integrating, or troubleshooting
cert-manager and its Helm chart. Start with the breaking-change checks below,
then open the topic reference that matches the resource or controller being
changed.

## Reference index

| Reference | Topics |
|---|---|
| [upgrades-and-support.md](references/upgrades-and-support.md) | Upgrade hazards, distribution changes, support policy, Kubernetes and OpenShift compatibility |
| [certificates-and-issuance.md](references/certificates-and-issuance.md) | Certificate defaults, renewal, keystores, name constraints, output, signing, reconciliation safety |
| [acme-and-dns-solvers.md](references/acme-and-dns-solvers.md) | ACME profiles and renewal information, HTTP-01, DNS-01, solver resources, retries |
| [gateway-ingress-and-shim.md](references/gateway-ingress-and-shim.md) | Gateway API, ListenerSet, generated Certificates, annotations, listener selection |
| [helm-operations-and-security.md](references/helm-operations-and-security.md) | Helm values, RBAC, NetworkPolicy, scheduling, identity, runtime classes, startup cleanup |
| [issuers-integrations-and-api.md](references/issuers-integrations-and-api.md) | Vault, Venafi, Azure DNS, issuer queries, server-side apply, API removals |
| [observability-and-reliability.md](references/observability-and-reliability.md) | Metrics, structured logs, cainjector, webhook recovery, controller reliability |

## Upgrade checks first

### Preserve private keys only when intentional

`Certificate.spec.privateKey.rotationPolicy` defaults to `Always`. Before an
upgrade, set it explicitly to `Never` on Certificates whose consumers cannot
accept a new key. The feature gate that once restored the old default is no
longer configurable, so per-Certificate policy is the durable control.

```yaml
spec:
  privateKey:
    rotationPolicy: Never
```

### Account for bounded revision history

An omitted `Certificate.spec.revisionHistoryLimit` defaults to `1`. Set an
explicit larger value where retaining more CertificateRequests is required for
operations or audit workflows.

### Remove obsolete configuration before upgrading

- Stop manually enabling the deprecated `ValidateCAA` gate.
- Replace `enableGatewayAPI` and `enableGatewayAPIListenerSet` with
  `gatewayAPI.enabled` and `gatewayAPI.enableListenerSet`.
- Remove `prometheus.servicemonitor.targetPort`,
  `prometheus.servicemonitor.path`, and `prometheus.podmonitor.path`; current
  monitoring uses `/metrics` and the `http-metrics` port name.
- Do not depend on `global.rbac.disableHTTPChallengesRole`; it was withdrawn
  after 1.18.0.
- Migrate integrations from the removed `ObjectReference` API type.
- Do not try to disable GA private-key rotation defaults or GA cainjector
  merging. Those behaviors are unconditional.

### Recreate controller token RBAC when needed

The chart no longer creates the Role and RoleBinding that allow the controller
to mint tokens for its own ServiceAccount. If an issuer's
`serviceAccountRef.name` selects that account, create explicit token RBAC or
move the issuer to a dedicated ServiceAccount before upgrading. Vault
Kubernetes authentication and Route53 configurations are common cases.

### Update dashboards and direct-resource automation

ACME request metrics use the bounded `action` label instead of `path`. Rewrite
queries and alerts, or use relabeling or recording rules when old
high-cardinality semantics truly must be preserved.

Starting in 1.19.6, the aggregate `cert-manager-edit` ClusterRole cannot create
Challenges or create, patch, or update Orders. Certificate-driven issuance is
unchanged; automation that directly manages these internal resources needs
explicit RBAC.

### Plan for workload identity changes

Default container UID and GID are both `65532`. Review admission controls,
security policies, and writable volume ownership that assumed UID `1000` or
GID `0`.

## High-value configuration

### Use safe release patch levels

- Avoid 1.19.0: use 1.19.1 or later to avoid persisted issuer-reference
  defaults and restored trailing-dot DNS SAN handling.
- Use 1.20.1 or later on OpenShift and when combining ListenerSet issuer
  configuration with Certificate parent-reference overrides.
- Use 1.20.2 or later when setting both `webhook.config` and
  `webhook.volumes`.
- Use 1.17.1 or later for Cloudflare DNS-01, and 1.17.4 or later for URI name
  constraints.

See [upgrades-and-support.md](references/upgrades-and-support.md) for all patch
level cautions and the active support policy.

### Configure HTTP-01 ingress selection precisely

Choose exactly one of `class`, `ingressClassName`, or `name` in each HTTP-01
solver. On versions using exact challenge paths, ingress-nginx strict path
validation may reject solver Ingresses. From 1.18.1, the compatibility gate can
restore the earlier path type:

```yaml
config:
  featureGates:
    ACMEHTTP01IngressPathTypeExact: false
```

Alternatively disable ingress-nginx `strict-validate-path-type` or use a fixed
ingress-nginx release. Per-Ingress solver class overrides use:

```yaml
metadata:
  annotations:
    acme.cert-manager.io/http01-ingress-ingressclassname: nginx
```

### Make renewal behavior explicit

Use `renewBefore`, `renewBeforePercentage`, or the newer `renewalPolicies`
according to the scheduling semantics required. The percentage calculation now
follows its specification and handles durations longer than roughly three
years, so an upgrade can move previously calculated renewal times.

For ACME servers that implement RFC 9773, enable `ACMEUseARI` to consume the
CA's recommended renewal window. Treat `waitInsteadOfSelfCheck` as an escape
hatch for split-horizon DNS or NAT hairpin environments, not a routine solver
default.

### Preserve CA overlap during rotation

Cainjector merges new CA certificates into injected bundles. In 1.19 the
`CAInjectorMerging` beta gate was enabled by default; it is now GA and cannot be
disabled. Design consumers to tolerate overlapping trust anchors during
rotation.

### Select modern PKCS#12 output when FIPS is required

Use the `Modern2026` PKCS#12 profile for AES-256 encryption and SHA-256 KDFs
compatible with FIPS 140-3 requirements. Literal JKS and PKCS#12 passwords are
supported but are mutually exclusive with `passwordSecretRef` and provide
compatibility, not meaningful keystore secrecy.

### Size and isolate solver workloads

Set HTTP-01 Pod requests and limits in an Issuer or ClusterIssuer when a
solver needs values different from the controller-wide flags. Runtime classes
can be assigned to both installed components and solver Pods, and the chart's
`global.commonLabels` can reach dynamic solver resources through
`--acme-http01-solver-extra-labels`.

## Task workflow

1. Identify the installed cert-manager chart and application patch version,
   Kubernetes or OpenShift version, enabled feature gates, and active Gateway
   API integrations.
2. Read [upgrades-and-support.md](references/upgrades-and-support.md) before
   changing minor versions or relying on a particular distribution channel.
3. Open the references for every affected area. A Certificate change often
   also requires the ACME, ingress, issuer, and observability references.
4. Compare omitted fields with current defaults, especially private-key
   rotation, revision history, feature gates, Pod identities, and chart-created
   RBAC.
5. Render Helm templates and validate CRDs before applying. Test generated
   solver resources, NetworkPolicies, monitoring selectors, and admission
   rules in a non-production namespace.
6. After rollout, inspect Certificate, CertificateRequest, Order, and Challenge
   conditions and events; then check request, challenge, validity, and
   expiration metrics.

## Configuration principles

- Prefer explicit per-resource settings when a changed default affects key
  rotation, renewal, solver sizing, or ingress selection.
- Treat feature gates according to their lifecycle: experimental gates require
  opt-in, beta gates may be enabled by default, and GA behaviors may no longer
  expose a gate.
- Use Certificate-driven issuance unless direct Order or Challenge management
  is essential and has dedicated RBAC.
- Validate credentials and endpoints before declaring an Issuer ready. Pay
  special attention to Vault audiences and paths, Azure tenants and zone type,
  Venafi authentication conditions, and DNS Secret validation.
- Prefer bounded-cardinality metrics and event details over literal whole-line
  log matching.
- Pin fixed patch releases where a known regression affects the chosen feature;
  do not assume the first patch of a minor line is safe for every integration.
