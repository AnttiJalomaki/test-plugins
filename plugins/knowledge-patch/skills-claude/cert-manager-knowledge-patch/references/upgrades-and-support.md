# Upgrades and Support

Use this reference to choose a safe patch level, migrate an installation, and
decide whether a cluster and cert-manager branch are supported.

## Distribution and installation changes

### OperatorHub publication ended

The Red Hat OpenShift and community OperatorHub catalogs stop at cert-manager
1.16.5. An OperatorHub-managed installation needs another distribution method
before moving to 1.17 or later.

### Upgrade configuration removals and defaults

Before upgrading, audit these cross-cutting changes:

- In 1.18, `Certificate.spec.privateKey.rotationPolicy` changed from `Never`
  to `Always`, and `Certificate.spec.revisionHistoryLimit` changed from `nil`
  to `1`. Set either field explicitly when the prior behavior is required.
- In 1.20, the default container UID changed from `1000` to `65532`, and the
  default GID changed from `0` to `65532`.
- In 1.21, chart-managed token-creation RBAC for the controller ServiceAccount
  was removed. Supply explicit RBAC for issuer configurations that reference
  that ServiceAccount.
- In 1.21, the ServiceMonitor `targetPort` and `path` overrides and PodMonitor
  `path` override were removed. Delete those values before Helm schema
  validation; metrics now use `/metrics` on the `http-metrics` port name.

The detailed migrations live in the certificate, Helm, and observability
references.

## Patch-level cautions

### 1.17 fixes

- A Cloudflare API change broke DNS-01 issuance until 1.17.1. Use 1.17.1 or
  later for Cloudflare.
- Releases before 1.17.4 copied permitted URI domains into the excluded URI
  domains of a CSR. Use 1.17.4 or later for URI name constraints.

### 1.18 fixes and withdrawn configuration

- HTTP-01 solver Ingresses use `PathType: Exact`. Starting in 1.18.1, the
  `ACMEHTTP01IngressPathTypeExact` feature gate can restore
  `ImplementationSpecific` paths for ingress compatibility.
- `global.rbac.disableHTTPChallengesRole` existed in 1.18.0 but was removed in
  1.18.2 because it was defective. It is unavailable for the remainder of the
  1.18 line.
- Starting in 1.18.3, larger PEM certificates and chains can be parsed.
- Starting in 1.18.5, HTTP-01 supports IPv6 subjects and public-key mismatches
  in issuer responses fail safely with backoff.

### Skip 1.19.0

CRD-level defaults for issuer-reference group and kind in 1.19.0 could cause
unnecessary reissuance. They were reverted in 1.19.1, so omitted fields again
use runtime defaults instead of being persisted as API defaults.

A dependency change in 1.19.0 also rejected trailing-dot DNS names in X.509 SAN
fields. Support was restored in 1.19.1. Use 1.19.1 or later for both cases.

The chart's `global.nodeSelector` should also be used with 1.19.2 or later,
which correctly merges the common selector with component-specific settings.

### 1.20 fixes

- In 1.20.0, combining ListenerSet issuer configuration with Certificate
  annotation overrides could create duplicate `parentRef` entries. Use 1.20.1
  or later for that combination.
- In 1.20.0, missing issuer-finalizer RBAC for the Order controller caused an
  OpenShift regression. Use 1.20.1 or later on OpenShift.
- Releases before 1.20.2 can render invalid Helm YAML when both
  `webhook.config` and `webhook.volumes` are set. Use 1.20.2 or later for that
  configuration.

## Support lifecycle

### Release-driven window

Each minor release is supported at least until the second subsequent minor is
released. Only the latest patch release on each supported branch receives
support. At this snapshot, 1.21 is supported until 1.23 and 1.20 until 1.22;
1.19 and earlier are end of life. The 1.22 release is tentatively planned for
November 2026.

### Kubernetes and OpenShift matrix

| cert-manager | Supported and tested Kubernetes | Supported OpenShift |
|:---:|:---:|:---:|
| 1.21 | 1.33-1.36 | 4.20-4.22 |
| 1.20 | 1.32-1.35 | 4.19-4.21 |

OpenShift support follows each release's mapped Kubernetes version. A mapping
for an OpenShift version that has not shipped yet can be predictive rather than
observed.

### Supported versus regularly tested

A Kubernetes version can be supported without receiving regular end-to-end
runs. Maintainers still respond to and fix reported problems on such a
version. Versions outside the supported range are generally neither tested nor
fixed.

### Backport policy

Security issues are backported to both supported releases and immediately
produce a patch release. Critical regressions and upgrade bugs are also usually
backported immediately. A long-standing fix, or a change with possible runtime
impact, can be withheld from patch branches when its stability risk is too
high.

### No upstream LTS branch

The maintainers do not provide LTS releases or updates after end of life.
CyberArk separately offers a commercial 1.17 LTS release through February 3,
2027.

## Source batch attribution

This topic incorporates the included batch identifiers `upgrade-1.17`,
`1.17`, `upgrade-1.18`, `1.18`, `upgrade-1.19`, `1.19`, `1.20`,
`upgrade-1.21`, `1.21`, and `support-lifecycle`.
