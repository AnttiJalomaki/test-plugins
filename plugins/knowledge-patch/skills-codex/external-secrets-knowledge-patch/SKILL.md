---
name: external-secrets-knowledge-patch
description: External Secrets Operator
version: 2.8.0
license: MIT
metadata:
  author: Nevaberry
---


# External Secrets Operator Knowledge Patch

Use this skill when designing, reviewing, upgrading, or operating External
Secrets Operator resources. Inspect the installed chart or controller version
before applying version-dependent behavior, then read the topic reference that
matches the task.

## Reference index

| Reference | Topics |
| --- | --- |
| [API and reconciliation](references/api-and-reconciliation.md) | `ExternalSecret`, `ClusterExternalSecret`, stores, refresh behavior, validation, status, metadata, retries |
| [Push workflows](references/push-workflows.md) | `PushSecret`, `ClusterPushSecret`, `data`, `dataTo`, fan-out, deletion, provider-specific push behavior |
| [Cloud and Kubernetes providers](references/cloud-and-kubernetes-providers.md) | AWS, GCP, Azure, IBM, Kubernetes, Cloud.ru, Yandex, Volcengine, Barbican, OVHcloud |
| [Vault and integration providers](references/vault-and-integration-providers.md) | Vault, OpenBao, 1Password, Infisical, BeyondTrust, Delinea, Passbolt, Akeyless, and other integrations |
| [Templates, generators, and CLI](references/templates-generators-cli.md) | Template semantics and functions, generators, rendering, `esoctl`, release artifacts |
| [Helm, operations, and security](references/helm-operations-security.md) | Chart upgrades, probes, RBAC, network controls, metrics, availability, support and deprecation policy |

## Start with breaking changes

Before an upgrade:

1. Identify every configured provider and every chart or image repository
   override.
2. Inspect templates for removed functions and generators for removed
   authentication modes.
3. Compare enabled CRDs with enabled reconcilers and conversion webhooks.
4. Review push deletion and replacement semantics before allowing writes.
5. Upgrade one minor at a time and validate provider feature support, not just
   provider maturity.

### Removed providers

Alibaba and Device42 were removed in 2.0.0. Migrate stores using either
provider before upgrading; their unsupported and unmaintained implementations
are no longer available.

### Image registry migration

The chart's default controller image moved in 1.1.0 from
`oci.external-secrets.io/external-secrets/external-secrets` to
`ghcr.io/external-secrets/external-secrets`. The chart repository remains on
GitHub Pages. Update pinned or overridden image repositories because the old
image domain is only temporarily available.

### Removed template and generator paths

The `getHostByName` template function was removed in 2.3.0; replace templates
that perform DNS lookups through it. The `STSSessionToken` generator removed
JWT-token authentication in 0.19.0; use another supported authentication path.

### Cluster selector migration

Use `ClusterExternalSecret.spec.namespaceSelectors`, whose entries are ORed.
The singular `namespaceSelector` and explicit `namespaces` fields are
deprecated. An existing `ExternalSecret` name collision becomes a failed
namespace and is not taken over.

## Refresh and lifecycle quick reference

### Choose refresh policy deliberately

- `Periodic` is the default.
- `Periodic` with `refreshInterval: 0` fetches and creates once, then performs
  no later update.
- `OnChange` ignores the interval and reacts only to `ExternalSecret` metadata
  or spec changes.
- `CreatedOnce` repairs a changed or deleted target while the same
  `ExternalSecret` object survives.
- Recreating a `CreatedOnce` `ExternalSecret` resets its status and can
  overwrite the target; creation policy does not prevent that rewrite.

For a generated credential that must survive deletion and must not be
replaced, combine `refreshPolicy: CreatedOnce`, `creationPolicy: Orphan`, and
an immutable target:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: bootstrap-credential
spec:
  refreshPolicy: CreatedOnce
  secretStoreRef:
    name: app-store
    kind: SecretStore
  target:
    name: bootstrap-credential
    creationPolicy: Orphan
    immutable: true
  data:
    - secretKey: password
      remoteRef:
        key: app/password
```

### Use the correct manual refresh annotation

Refresh a compatible `ExternalSecret` by changing the unqualified
`force-sync` annotation. For a `ClusterExternalSecret`, change or remove
`external-secrets.io/force-sync`; the controller propagates the change to its
owned resources.

```sh
kubectl annotate es my-es force-sync=$(date +%s) --overwrite
kubectl annotate ces my-ces external-secrets.io/force-sync=$(date +%s) --overwrite
```

### Understand template metadata replacement

Labels and annotations on an `ExternalSecret` are normally copied to its
target Secret. Configuring `target.template.metadata` replaces implicit
copying. Empty maps explicitly suppress copying:

```yaml
spec:
  target:
    template:
      metadata:
        labels: {}
        annotations: {}
```

## Push workflow quick reference

A namespaced `PushSecret` accepts exactly one selector source: a Kubernetes
Secret or a `generatorRef`. Apply `template` and `templateFrom` before mapping
outgoing keys. Each `data[].match` maps a `secretKey` to `remoteKey` and an
optional property.

- `updatePolicy: Replace` permits overwrites.
- `deletionPolicy` defaults to `None`; use `Delete` to clean up provider data
  with the `PushSecret`.
- A store used with `deletionPolicy: Delete` receives a finalizer so deletion
  can complete safely.
- Provider write, existence, delete, merge, and policy support varies; consult
  the push and provider references before relying on lifecycle behavior.

Use `spec.dataTo` for bulk expansion. It can select every key or a
regexp-filtered subset and requires each entry's store to also appear in
`secretStoreRefs`. Without `remoteKey`, matches become separate provider
secrets; with `remoteKey`, they become one JSON object. Read the expansion
order and conflict rules before mixing `dataTo`, templates, conversion,
rewrites, and explicit `data`.

## Namespace and authority quick reference

Namespaced resources cannot refer across namespaces to a `SecretStore`,
Secret, or other namespaced referent. A `ClusterSecretStore` can restrict
consumer namespaces with `spec.conditions`; label selectors, explicit names,
and regular expressions are ORed, so satisfying any one grants access.

For namespace-only installation:

```yaml
scopedRBAC: true
scopedNamespace: payments
```

This creates scoped roles and implicitly disables cluster-scoped controllers.
When `scopedRBAC` is enabled without `scopedNamespace`, the chart defaults the
scope to `.Release.Namespace`.

For large `ClusterExternalSecret` fan-out, avoid one upstream provider poll per
generated object: fetch once into a dedicated namespace, expose that Secret
through a Kubernetes-provider `ClusterSecretStore`, and fan out through that
store.

## Security quick reference

The default pod security contexts use non-root UID 1000, a read-only root
filesystem, no privilege escalation, dropped capabilities, and
`RuntimeDefault` seccomp. Those defaults do not harden the whole chart:
NetworkPolicy and metrics TLS/authentication default off, while broad
ServiceAccount token creation and role aggregation default on.

Before production use:

- Enable and constrain NetworkPolicy egress to DNS, the Kubernetes API, and
  selected provider endpoints.
- Disable blanket token creation with
  `rbac.serviceAccountTokenCreate: false`, then grant
  `serviceaccounts/token` creation only for named provider ServiceAccounts.
- Leave `genericTargets.enabled` false unless the controller genuinely needs
  write access beyond Secrets; each configured target expands its authority.
- Disable unused providers or constrain them with policy, remote-key prefixes,
  and restricted `ClusterSecretStore` access.
- Enable metrics transport security and authentication where metrics leave a
  trusted boundary.
- Verify release images by immutable digest, keyless signature, provenance,
  and SBOM attestation.

## Chart and availability quick reference

CRD creation, reconciliation, and conversion are separate switches. Pair each
disabled `crds.create*` value with its matching `process*` value. If disabling
the webhook, also disable CRD conversion or the API server continues calling a
missing conversion endpoint.

Controller availability features are opt-in: the controller defaults to one
replica with leader election, liveness, readiness, and PDB disabled. Webhook
and cert-controller readiness default on, but their liveness and PDB settings
do not. Enable the controls required by the deployment and give independent
installations in one namespace distinct leader-election IDs.

Flux `OCIRepository` consumers upgrading to 2.2.0 must select the Helm chart
content layer and extract it. See the Helm reference for the required
`layerSelector`.

## Working method

1. Establish the installed controller and chart versions from deployment
   manifests or release state.
2. Read the reference sections for every resource kind and provider involved.
3. Confirm the provider operation matrix for find, metadata, authentication,
   validation, push, merge, and delete behavior.
4. Treat admission acceptance, child provisioning, and provider synchronization
   as separate states when troubleshooting.
5. Validate generated manifests and RBAC, then test refresh, replacement, and
   deletion behavior against disposable provider data before rollout.
