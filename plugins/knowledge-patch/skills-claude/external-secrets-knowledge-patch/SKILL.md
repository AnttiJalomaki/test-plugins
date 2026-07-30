---
name: external-secrets-knowledge-patch
description: External Secrets Operator
version: 2.8.0
license: MIT
metadata:
  author: Nevaberry
---


# External Secrets Operator Knowledge Patch

Use this skill when designing, reviewing, upgrading, or troubleshooting External
Secrets Operator (ESO) resources, providers, Helm installations, generated
credentials, and secret-push workflows.

## Start with project reality

1. Read the chart or controller version from the deployment manifest, Helm lock,
   GitOps source, or image digest.
2. Read the installed CRDs before proposing fields; controller and CRD versions
   can drift independently.
3. Identify the resource kind and API version, store scope, provider, refresh
   policy, creation/deletion policies, and controller feature gates.
4. Inspect status conditions and warning events on the resource and its store.
5. Prefer the live schema, rendered chart, controller flags, and observed behavior
   when they disagree with compatibility guidance.
6. If the installed release is newer than this skill, state that the guidance may
   be stale and verify behavior against the deployed artifacts.

## Reference index

| Reference | Topics |
| --- | --- |
| [references/api-and-reconciliation.md](references/api-and-reconciliation.md) | v1 resource semantics, refresh, selectors, status, validation, metadata, reconciliation |
| [references/push-secrets.md](references/push-secrets.md) | PushSecret and ClusterPushSecret mapping, `dataTo`, lifecycle, provider write behavior |
| [references/cloud-and-vault-providers.md](references/cloud-and-vault-providers.md) | AWS, GCP, Azure, Vault, OpenBao, Kubernetes, IBM, Yandex, Volcengine |
| [references/integration-providers.md](references/integration-providers.md) | 1Password, Akeyless, Infisical, Passbolt, Delinea, BeyondTrust, and other providers |
| [references/templates-generators-and-cli.md](references/templates-generators-and-cli.md) | templates, conversions, generators, `esoctl`, output and release artifacts |
| [references/helm-and-operations.md](references/helm-and-operations.md) | chart migration, CRD/controller switches, probes, HA, metrics, RBAC, networking |
| [references/security-and-support.md](references/security-and-support.md) | support policy, deprecation scope, provider maturity, pod hardening, artifact verification |

## Breaking changes and migrations

### Remove unsupported providers before a 2.x upgrade

Alibaba and Device42 providers were removed in 2.0.0. Find every store using
either provider and migrate it before changing the controller or CRDs. A CRD
upgrade does not provide a compatibility shim for removed provider blocks.

### Move pinned images to GHCR

The default image repository moved from
`oci.external-secrets.io/external-secrets/external-secrets` to
`ghcr.io/external-secrets/external-secrets`. Update explicit repository
overrides; the old image domain is only temporarily available. The chart
repository itself remains on GitHub Pages.

### Remove obsolete template and authentication options

- Replace templates that call `getHostByName`; DNS lookup through that function
  is no longer available.
- Replace JWT-token authentication on the `STSSessionToken` generator with a
  supported authentication path.
- Treat the old beta API serving switch only as migration aid. Prefer
  `external-secrets.io/v1` resources and verify the installed conversion setup.

### Flux OCI chart upgrades need layer extraction

When Flux fetches the chart with `OCIRepository`, select the Helm content layer
and extract it. Without the layer selector, a chart upgrade can fail before ESO
resources are reconciled.

```yaml
spec:
  url: oci://ghcr.io/external-secrets/charts/external-secrets
  layerSelector:
    mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip
    operation: extract
```

## ExternalSecret quick reference

### Choose refresh behavior deliberately

- `Periodic` is the default. With `refreshInterval: 0`, ESO fetches and creates
  once but does not perform later periodic updates.
- `OnChange` ignores the interval and reconciles secret data only after the
  `ExternalSecret` metadata or spec changes.
- `CreatedOnce` repairs a changed or deleted target while the source object and
  its status survive. Recreating the `ExternalSecret` resets that status and may
  overwrite an existing target.
- Use `force-sync` on an `ExternalSecret`; use
  `external-secrets.io/force-sync` on a `ClusterExternalSecret`.
- Sync windows can gate periodic refreshes. Account for them before diagnosing a
  delayed update as a provider failure.

For a generated value that must survive deletion and must never be replaced,
combine `refreshPolicy: CreatedOnce`, `creationPolicy: Orphan`, and an immutable
target. Creation policy alone does not prevent a rewrite after the
`ExternalSecret` is recreated.

### Understand target creation and metadata

`CreateOrMerge` is an accepted target creation policy. Target template labels
and annotations replace implicit copying from the `ExternalSecret`; empty maps
explicitly suppress copying. Templates can add finalizers, and target
`objectMeta` and `ownerReferences` propagate to generated resources.

### Fan out without multiplying provider calls

`ClusterExternalSecret.spec.namespaceSelectors` is an ORed list. The singular
selector and explicit `namespaces` field are deprecated. Each generated child
normally polls the upstream provider independently. For large fan-out, fetch
once into a dedicated namespace and replicate through a Kubernetes-provider
`ClusterSecretStore`.

Name collisions are reported as failed namespaces rather than adopted. A
`ClusterSecretStore` may restrict access with ORed label, explicit-name, and
regular-expression conditions.

## PushSecret quick reference

### Map one source into remote targets

A namespaced `PushSecret` selects exactly one Kubernetes Secret or one
`generatorRef`. `template` and `templateFrom` transform outgoing values before
`data[].match` maps each `secretKey` to a `remoteKey` and optional property.
`updatePolicy: Replace` permits overwrite. `deletionPolicy` defaults to `None`;
set it to `Delete` for remote cleanup.

`ClusterPushSecret` fans its embedded spec into namespaces matched by any
`namespaceSelectors` entry. Its `Ready` condition reports child provisioning,
not remote-provider sync; inspect each generated `PushSecret` for sync errors.

### Use `dataTo` for bulk expansion

`spec.dataTo` expands all selected source keys or a regexp-matched subset. Every
entry needs a `storeRef`, and that store must also be in `secretStoreRefs`.
Without `remoteKey`, each match becomes its own provider secret; with
`remoteKey`, all matches become one JSON object.

Expansion applies `spec.template` first, then key conversion, matching, and
rewriting. Explicit `spec.data` wins for the same original key. Invalid regexps
fail; no matches succeed as a no-op; duplicate remote keys fail. Per-key mode
supports regexp and template rewrites but not merge, while bundle mode performs
no rewrites. Lifecycle policies apply to every expanded target.

### Verify provider write semantics

Provider maturity does not imply push, merge, or delete support. In particular:

- Kubernetes pushes replace the entire remote Secret rather than merging keys.
- Conjur returns explicit errors for unsupported push and delete operations.
- A store used by a deleting `PushSecret` receives a finalizer so cleanup can
  complete.
- Cross-namespace pushes through `ClusterSecretStore` are supported, but
  namespaced reference rules and store conditions still apply.

## Templates and generators quick reference

- Template values preserve native values in value scope. Values from
  `templateFrom` are decoded before use.
- Custom delimiters avoid collisions with embedded template languages.
- `certSANs` extracts certificate subject alternative names; `hexdec` converts
  hexadecimal input to decimal; slice notation is resolved correctly.
- Generic target paths preserve mixed-case components.
- JSONPath under `dataFrom` can be templated, and source null-byte handling is
  configurable.
- Available generator families include registry credentials, Grafana service
  accounts, MFA tokens, SSH keys, bootstrap generators, and GitLab deploy
  tokens. Validate `generatorRef` type and provider-specific authentication.
- Use `esoctl` for template-data/secret rendering and bootstrap-generator
  commands.

## Installation and operations quick reference

### Keep CRDs and reconcilers aligned

Disabling CRD creation does not disable its reconciler. Pair every disabled
`crds.create*` option with the corresponding `process*` option. If the webhook
is disabled, disable CRD conversion too or the API server calls a nonexistent
conversion endpoint.

For a namespace-only installation, set `scopedRBAC: true` and set
`scopedNamespace` (it defaults to the Helm release namespace when omitted).
This implicitly disables cluster-scoped controllers.

### Opt in to availability and secure serving

The controller defaults to one replica; its leader election, liveness,
readiness, and PodDisruptionBudget controls require deliberate configuration.
Webhook and cert-controller readiness are enabled, but their liveness and PDBs
are not. Give independent installations in one namespace distinct leader lease
IDs and tune lease timing only with an HA plan.

Metrics TLS, authentication, and authorization are configurable but are not a
substitute for NetworkPolicy. Permit only Kubernetes API, DNS, and required
provider endpoints. Account for controller metrics/health, webhook admission
and metrics/health, and cert-controller metrics/health ports.

### Minimize controller authority

- Disable blanket ServiceAccount token creation and grant
  `serviceaccounts/token` only for named provider accounts where practical.
- Keep generic targets disabled unless required; each additional resource type
  expands create, update, and delete authority.
- Disable unused cluster controllers and providers; provider build tags can
  remove unwanted providers in custom builds.
- Control aggregation into the admin role and review view/edit aggregation.
- Treat optional chart NetworkPolicy as a starting point, not a complete
  exfiltration policy.

## Troubleshooting order

1. Confirm resource and store `apiVersion`, installed CRD schema, and controller
   image actually match the intended deployment.
2. Inspect the resource status, store status, warning events, and generated
   child object. An unknown `SecretStore` status is distinct from a known
   failure.
3. Check refresh policy, sync windows, refresh interval, manual annotation name,
   and the less aggressive failed-reconcile cadence.
4. Validate namespace boundaries, `ClusterSecretStore.spec.conditions`, referent
   namespaces, TokenRequest RBAC, CA providers, and provider authentication.
5. For push failures, check store write capability, existence checks, update and
   deletion policies, remote-key collisions, and provider replacement semantics.
6. For chart problems, render Helm values and verify CRD/reconciler, webhook/
   conversion, probe, PDB, image registry, OCI layer, and RBAC combinations.
7. For provider errors, consult the provider-specific references before assuming
   a common feature set; diagnostics and absence semantics differ by provider.
