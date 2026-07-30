# Upgrades and Support

Use this reference before selecting a release, changing minor versions, or
deciding whether a problem should be fixed by upgrading to a later patch.

Versioned source attribution for this topic set: `upgrade-1.17`, `1.17`,
`upgrade-1.18`, `1.18`, `upgrade-1.19`, `1.19`, `1.20`, `upgrade-1.21`, and
`1.21`.

## Upgrade-sensitive defaults

### Private-key rotation and revision history

In 1.18, `Certificate.spec.privateKey.rotationPolicy` defaults to `Always`
instead of `Never`. Before upgrading, explicitly set `Never` on Certificates
that must preserve their old keys. In 1.20,
`DefaultPrivateKeyRotationPolicyAlways` became GA and ceased to be a
configurable feature gate; the per-Certificate field is the way to override the
default.

Also in 1.18, `Certificate.spec.revisionHistoryLimit` defaults to `1` instead
of `nil`. A resource that omits the field begins using that limit after the
upgrade.

### Renewal calculations

The 1.17 calculation fix for `renewBeforePercentage` can change the renewal
time of existing Certificates. In 1.21, the same percentage calculation was
fixed for durations longer than roughly three years; older behavior could
reject such values or compute the wrong time. Review scheduled renewal times
after either transition.

### Cryptographic hashes for larger RSA keys

From the 1.17 upgrade, 3072-bit RSA certificates use SHA-384 and 4096-bit RSA
certificates use SHA-512. If rotation causes consumer failures, verify support
for the stronger signature hash.

## Required upgrade preparation

### Feature gates

- `ValidateCAA` is deprecated and was scheduled for removal in 1.18. Stop
  enabling it manually.
- `NameConstraints` and `UseDomainQualifiedFinalizer` are beta and enabled by
  default in 1.17.
- `AdditionalCertificateOutputFormats` is GA in 1.18 and no longer needs a
  gate.
- `OtherNames` is beta and enabled by default in 1.20.
- `CAInjectorMerging` progressed from opt-in in 1.17, to beta and enabled by
  default in 1.19, to GA and unconditional in 1.21. In 1.19 it could still be
  disabled to obtain replacement semantics; in 1.21 it cannot.
- Cainjector server-side apply is unconditional in 1.21, and the
  `ServerSideApply` feature gate is deprecated.

### 1.21 chart removals

The chart no longer creates the Role and RoleBinding that allow the controller
to create tokens for its own ServiceAccount. Before upgrading, find issuers
whose `serviceAccountRef.name` selects that account. Create the token-request
RBAC explicitly or move the issuer to a dedicated ServiceAccount with its own
RBAC. Vault Kubernetes authentication and Route53 are examples that can depend
on this setup.

Remove `prometheus.servicemonitor.targetPort`,
`prometheus.servicemonitor.path`, and `prometheus.podmonitor.path`; chart schema
validation rejects them. Metrics use the fixed `/metrics` path and the
`http-metrics` port name. Custom scrape configuration must also stop referring
to `tcp-prometheus-servicemonitor`.

### API and RBAC consumers

The deprecated `ObjectReference` type is removed in 1.21. Rebuild integrations
against a current reference type before upgrading.

From 1.19.6, the aggregate `cert-manager-edit` ClusterRole does not permit
creating Challenges or creating, patching, or updating Orders. Normal
Certificate-driven issuance does not need those permissions. Grant explicit,
narrow permissions only to tooling that directly manages these internal
objects.

The `global.rbac.disableHTTPChallengesRole` value appeared in 1.18.0 but was
withdrawn in 1.18.2 because it was buggy. It is not a supported setting for the
rest of the 1.18 series.

## Patch-release selection

### 1.17 fixes

- Use at least 1.17.1 for Cloudflare DNS-01 issuance after a Cloudflare API
  change broke earlier behavior.
- Starting with 1.17.3, ACME challenge authorization has a two-minute timeout,
  reducing premature `error waiting for authorization` failures.
- Use at least 1.17.4 for URI name constraints. Earlier 1.17 versions copied
  permitted URI domains into the CSR's excluded URI domains.

### 1.18 fixes and compatibility

HTTP-01 solver Ingresses use `PathType: Exact`. With ingress-nginx strict path
validation, use cert-manager 1.18.1 or later and disable
`ACMEHTTP01IngressPathTypeExact`, disable ingress-nginx
`strict-validate-path-type`, or run ingress-nginx 1.12.6+ or 1.13.2+.

Starting in 1.18.3, larger PEM certificates and chains are accepted, including
leaf certificates with many DNS names or other identities. Starting in 1.18.5,
HTTP-01 accepts IPv6 Host headers, and mismatched issuer responses are rejected
with backoff rather than causing an infinite reissuance loop.

### 1.19 fixes

Do not remain on 1.19.0. CRD defaults for `Certificate` and
`CertificateRequest` issuer-reference group and kind could persist defaulted
fields and trigger unnecessary reissuance. They were reverted in 1.19.1 so
omitted fields again receive runtime defaults.

The same 1.19.0 release rejected trailing-dot X.509 DNS SANs due to a dependency
change. Support returned in 1.19.1. Use 1.19.2 or later when relying on
`global.nodeSelector`; that release fixed merging with component selectors.

### 1.20 fixes

- 1.20.0 can generate duplicate Gateway `parentRef` entries when inferred
  issuer configuration and Certificate annotation overrides are combined. Use
  1.20.1 or later.
- 1.20.0 omitted issuer-finalizer RBAC needed by the Order controller on
  OpenShift. Use 1.20.1 or later.
- Releases before 1.20.2 can render invalid Helm YAML when both
  `webhook.config` and `webhook.volumes` are set. Use 1.20.2 or later for that
  combination.

## Distribution and lifecycle

### OperatorHub installations

Red Hat OpenShift and community OperatorHub catalogs stop at cert-manager
1.16.5. To move an OperatorHub installation to 1.17 or later, select another
distribution method rather than waiting for a newer catalog package.

### Support window

Each minor is supported at least until the second subsequent minor ships. Only
the latest patch on a supported branch receives support. At the
support-lifecycle snapshot, 1.21 is supported until 1.23, 1.20 until 1.22, and
1.19 and earlier are EOL; 1.22 was tentatively planned for November 2026.

The currently published compatibility matrix is:

| cert-manager | Supported and tested Kubernetes | Supported OpenShift |
|:------------:|:-------------------------------:|:-------------------:|
| 1.21 | 1.33-1.36 | 4.20-4.22 |
| 1.20 | 1.32-1.35 | 4.19-4.21 |

OpenShift follows the Kubernetes version mapped to each OpenShift release.
Mappings for unreleased OpenShift versions may be predictions.

A Kubernetes version can be supported without regular testing: maintainers
still respond to reports and fix accepted bugs, but do not run regular
end-to-end tests. Versions outside the supported range are generally neither
tested nor fixed.

Security fixes are backported to the two supported branches and trigger an
immediate patch. Critical regressions and upgrade bugs are also usually
backported promptly. Older fixes or changes with runtime risk may be withheld
from patch branches for stability.

Upstream has no LTS release and provides no updates after EOL. CyberArk offers
a commercial 1.17 LTS through February 3, 2027.
