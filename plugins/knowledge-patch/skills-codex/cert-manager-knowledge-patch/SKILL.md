---
name: cert-manager-knowledge-patch
description: cert-manager
version: 1.21
license: MIT
metadata:
  author: Nevaberry
---


# cert-manager Knowledge Patch

Use this skill when upgrading, installing, configuring, or debugging cert-manager,
its Helm chart, issuers, ACME solvers, Gateway API integration, cainjector, or
Certificate resources. Start with the upgrade hazards, then open the reference
whose topic matches the work.

## Reference index

| Reference | Topics |
| --- | --- |
| [Upgrades and support](references/upgrades-and-support.md) | Upgrade blockers, changed defaults, patch-release fixes, compatibility, support policy |
| [Certificates and renewal](references/certificates-and-renewal.md) | Certificate fields, private keys, keystores, renewal, name constraints, output formats |
| [ACME and challenge solvers](references/acme-and-challenge-solvers.md) | ACME profiles and ARI, HTTP-01, DNS-01, solver configuration, challenge behavior |
| [Issuers and external providers](references/issuers-and-external-providers.md) | Vault, Venafi, Azure DNS, Cloudflare, CloudDNS, DigitalOcean, issuer validation |
| [Gateway, Ingress, and cert-shim](references/gateway-ingress-and-cert-shim.md) | Gateway listeners, ListenerSets, annotations, generated Certificates, HTTPRoutes |
| [Helm and platform operations](references/helm-and-platform-operations.md) | Chart values, RBAC, NetworkPolicy, Pod identity, scheduling, runtime classes, OpenShift |
| [Observability, cainjector, and clients](references/observability-cainjector-and-clients.md) | Metrics, logs, CA injection, kubectl discovery, field selectors, apply clients |

## Upgrade hazards first

### Pin private-key behavior before a 1.18 upgrade

`Certificate.spec.privateKey.rotationPolicy` changed from `Never` to `Always`.
Set `rotationPolicy: Never` explicitly before upgrading any Certificate that must
keep its key. The `DefaultPrivateKeyRotationPolicyAlways` gate is GA and no
longer configurable in 1.20, so explicit per-Certificate policy is the durable
control.

```yaml
spec:
  privateKey:
    rotationPolicy: Never
```

Also account for `spec.revisionHistoryLimit` defaulting to `1` rather than
remaining unset in 1.18.

### Skip known-bad initial patch releases

- Use 1.17.1 or later for Cloudflare DNS-01, 1.17.4 or later for correct URI
  name constraints, and 1.17.3 or later for the ACME authorization timeout.
- Use 1.18.1 or a compatible ingress-nginx release when exact HTTP-01 paths
  conflict with strict path validation.
- Use 1.19.1 or later to avoid persisted issuer-reference defaults and to
  accept trailing-dot DNS SANs; use 1.19.2 or later for merged
  `global.nodeSelector` behavior.
- Use 1.20.1 or later on OpenShift and when ListenerSet parent-reference
  overrides are combined; use 1.20.2 or later with both `webhook.config` and
  `webhook.volumes`.

### Prepare for 1.21 chart removals

The chart no longer creates token-request RBAC for the controller's own
ServiceAccount. If an issuer names that account in `serviceAccountRef`, add the
Role and RoleBinding yourself or migrate the issuer to a dedicated account.

Remove these values before upgrading because chart schema validation rejects
them:

```yaml
prometheus:
  servicemonitor:
    targetPort: <removed>
    path: <removed>
  podmonitor:
    path: <removed>
```

Scrape `/metrics` through the `http-metrics` port name. Replace any dependency
on the old `tcp-prometheus-servicemonitor` Service port.

### Update integrations for removed or restricted APIs

- Migrate integrations away from the removed `ObjectReference` API type.
- Starting in 1.19.6, `cert-manager-edit` no longer grants direct creation of
  Challenges or creation, patching, and updating of Orders. Add narrowly scoped
  RBAC only for tooling that directly manages these internal resources.
- Do not configure the withdrawn
  `global.rbac.disableHTTPChallengesRole` value; it disappeared in 1.18.2.
- Stop enabling deprecated gates: `ValidateCAA` was scheduled for removal in
  1.18, cainjector merging is unconditional in 1.21, and cainjector's
  `ServerSideApply` gate is deprecated.

### Recheck operational assumptions

- Larger RSA keys now select stronger hashes: SHA-384 for 3072-bit keys and
  SHA-512 for 4096-bit keys. Verify all consumers before rotation.
- Structured log context can break complete-line and literal-string matches.
- The container UID and GID default to `65532`; update admission rules and
  volume ownership that assumed UID `1000` or GID `0`.
- ACME request metrics use bounded `action` labels instead of `path`; rewrite
  alerts and dashboards before the metric change reaches production.

## High-value Certificate controls

### Set renewal and retry behavior deliberately

Use `renewBefore` or `renewBeforePercentage` for the established renewal
controls and `renewalPolicies` for more expressive scheduling. Long-duration
percentage calculations are corrected in 1.21. Configure failed
CertificateRequest backoff with the controller flag, controller config, or
chart value:

```yaml
config:
  certificateRequestMaximumBackoffDuration: 8h
```

The default maximum exponential-backoff duration is 32 hours.

### Select compatible key and output formats

Signature algorithms are configurable. Additional certificate output formats
are always available without a gate. For FIPS 140-3-compatible PKCS#12 output,
select the `Modern2026` profile, which uses AES-256 and SHA-256 KDFs.

Generated JKS and PKCS#12 keystores can use a literal `password`, but it is
mutually exclusive with `passwordSecretRef` and provides software
compatibility, not additional keystore security.

### Observe certificate validity directly

Use these timestamp gauges for issuance and expiry monitoring:

```text
certmanager_certificate_not_before_timestamp_seconds
certmanager_certificate_not_after_timestamp_seconds
```

## High-value ACME controls

### Handle ingress path compatibility

HTTP-01 solver Ingresses use `PathType: Exact`. If ingress-nginx strict path
validation rejects them, use a fixed ingress-nginx version, disable its strict
validation, or from cert-manager 1.18.1 set:

```yaml
config:
  featureGates:
    ACMEHTTP01IngressPathTypeExact: false
```

Configure exactly one of `class`, `ingressClassName`, or `name` in a solver.
An Ingress can override `ingressClassName` with
`acme.cert-manager.io/http01-ingress-ingressclassname`.

### Use CA-directed or delayed validation when needed

Enable `ACMEUseARI` to experiment with RFC 9773 renewal information and
CA-recommended renewal windows. For split-horizon DNS or NAT hairpin cases,
`waitInsteadOfSelfCheck` can skip the local self-check, wait for the configured
duration, and ask the ACME server to validate directly.

### Configure solver resources close to the issuer

HTTP-01 Pod requests and limits may be set per Issuer or ClusterIssuer and
override the controller-wide solver resource flags. Runtime classes are
available for solver Pods, and RFC2136 DNS-01 can explicitly select its update
protocol.

## High-value Gateway and Ingress controls

- Passthrough Gateway TLS listeners are ignored.
- ListenerSet certificate generation is alpha and requires its feature gate.
- `cert-manager.io/ignore-tls-listeners` excludes selected Gateway listeners.
- A TLS-only ListenerSet can fall back to its parent Gateway's HTTP listener by
  setting `acme.cert-manager.io/http01-parentreffallback: "true"`.
- Cert-shim reads `cert-manager.io/alt-names` and
  `cert-manager.io/ip-sans` on ingress-like resources.
- Changing Duration or `RenewBefore` on an Ingress or Gateway reconciles the
  generated Certificate immediately.

Use the nested controller configuration; the flat fields remain accepted but
are deprecated:

```yaml
gatewayAPI:
  enabled: true
  enableListenerSet: true
```

## High-value Helm and cluster controls

- Use chart-managed NetworkPolicies for each Deployment when network isolation
  is required; default network policies include IPv6.
- Use `global.nodeSelector` for common scheduling, `global.hostUsers: false`
  for experimental user namespaces on Kubernetes 1.33+, and component or
  solver runtime-class settings when sandboxing Pods.
- Percentage values are accepted for PodDisruptionBudget availability.
- Annotation keys and values on chart-managed ServiceAccounts are templates,
  which supports workload-identity configuration derived from chart values.
- A parent chart can use `enabled` to toggle cert-manager as a dependency.

## Triage sequence

1. Identify the exact cert-manager patch release, installation method, and
   relevant Kubernetes or OpenShift version.
2. Check [upgrades and support](references/upgrades-and-support.md) for a known
   initial-patch regression or EOL branch.
3. Inspect the generated resource as well as its source Ingress, Gateway,
   ListenerSet, Certificate, or Issuer.
4. For ACME, inspect Challenge events and confirm that solver ingress selection
   is exclusive.
5. For issuer readiness, validate referenced Secrets, authentication audience,
   provider tenant or zone selection, and server-name/path validation.
6. For monitoring regressions, check metric labels, fixed scrape path and port,
   and chart-generated Service labels.
7. Prefer explicit resource fields over relying on a historical default or a
   feature gate that later became unconditional.
