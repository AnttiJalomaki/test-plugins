---
name: flux-cd-knowledge-patch
description: Flux CD
version: 2.9.0
license: MIT
metadata:
  author: Nevaberry
---


# Flux CD Knowledge Patch

Use this skill when upgrading, configuring, or troubleshooting Flux controllers,
APIs, authentication, reconciliation, artifacts, image automation, or event
delivery. Start with the upgrade checks because removed APIs and changed Helm
defaults can block reconciliation or alter release behavior.

## Reference index

| Reference | Topics |
| --- | --- |
| [upgrades-and-platforms.md](references/upgrades-and-platforms.md) | Required API migrations, breaking defaults, feature migrations, Kubernetes and OpenShift support |
| [kustomizations-and-helm.md](references/kustomizations-and-helm.md) | Health, dependencies, deletion, apply ordering, decryption, values, retries, inventory, config watches |
| [sources-auth-and-artifacts.md](references/sources-auth-and-artifacts.md) | Git and OCI sources, GitHub Apps, Workload Identity, signing, verification, ArtifactGenerator |
| [image-automation.md](references/image-automation.md) | Stable image APIs, digest pinning, providers, commit templates, sparse checkout, signing |
| [notifications-and-receivers.md](references/notifications-and-receivers.md) | Event metadata, commit status, providers, OpenTelemetry, pull-request comments, Receiver filtering and OIDC |
| [cli-and-operator.md](references/cli-and-operator.md) | Debug commands, artifact and plugin commands, Receiver triggering, Flux Operator Web UI |

## Upgrade checks

### Migrate stored APIs before installing new CRDs

Run the migration while the old APIs are still served:

```bash
flux migrate
```

Then verify that stored objects use APIs retained by the target release. The
critical removal boundaries are:

- Before Flux 2.7, migrate objects stored as the `v1beta1` source,
  kustomize, Helm, image, or notification APIs.
- Before Flux 2.8, migrate the remaining `v1beta2` source, kustomize, and Helm
  APIs.
- Before Flux 2.9, migrate `image.toolkit.fluxcd.io/v1beta2` and
  `notification.toolkit.fluxcd.io/v1beta2` objects.

See [upgrades-and-platforms.md](references/upgrades-and-platforms.md) for the
exact removed API groups and the GCR Receiver Secret requirement.

### Preserve Helm behavior deliberately

Helm v4 changes new releases to server-side apply and makes kstatus health
checking the default. Existing stored releases remain on client-side apply
until explicitly opted in. Enable `UseHelm3Defaults` when the earlier apply and
health behavior is required.

The HelmRelease post-render default later changes from `nohooks` to `combined`.
If hooks must bypass post-rendering, set the strategy explicitly to `nohooks`
before the upgrade.

### Update image automation manifests and templates

Use `image.toolkit.fluxcd.io/v1` for `ImageRepository`, `ImagePolicy`, and
`ImageUpdateAutomation`. Replace removed image-reflector-controller `autologin`
flags with `ImageRepository.spec.provider`. Replace commit-template uses of
`.Updated` and `.Changed.ImageResult` with `.Changed.FileChanges`,
`.Changed.Objects`, or `.Changed.Changes`.

## High-value configuration

### Reconcile when referenced configuration changes

Add this label to a referenced ConfigMap or Secret:

```yaml
metadata:
  labels:
    reconcile.fluxcd.io/watch: Enabled
```

It can trigger immediate reconciliation for Kustomization substitution,
decryption, and kubeConfig references; HelmRelease values and kubeConfig
references; and Receiver Secrets. At controller scope,
`--watch-configs-label-selector=owner!=helm` watches every matching reference
without requiring the per-object label.

### Extend health and dependency readiness with CEL

Teach a Kustomization how to evaluate a custom kind:

```yaml
spec:
  wait: true
  healthCheckExprs:
    - apiVersion: cluster.x-k8s.io/v1beta1
      kind: Cluster
      failed: "status.conditions.filter(e, e.type == 'Ready').all(e, e.status == 'False')"
      current: "status.conditions.filter(e, e.type == 'Ready').all(e, e.status == 'True')"
```

Dependency entries for both Kustomizations and HelmReleases can also use CEL
readiness expressions. A health expression can omit `kind` when it should
apply to every kind in an API group.

### Ignore fields owned by another controller

Use `Kustomization.spec.ignore` to exclude selected fields from drift detection
and apply:

```yaml
spec:
  ignore:
    - target:
        kind: Deployment
      paths:
        - /spec/replicas
```

This lets an HPA, for example, own `/spec/replicas` without Flux fighting it.

### Make failed Helm releases recover promptly

Use the `RetryOnFailure` install or upgrade strategy. If enabling
`CancelHealthCheckOnNewRevision` for helm-controller, also enable
`DefaultToRetryOnFailure`; otherwise a cancellation can leave a release stuck
under the no-retry default. A canceled check reports `HealthCheckCanceled` in
the `Ready` condition.

## Authentication and supply-chain choices

### Match registry providers to repository URLs

For `OCIRepository` and `ImageRepository`, use `aws`, `azure`, or `gcp` only
when the registry matches and automatic OIDC authentication is intended. For a
public repository or image-pull-secret authentication, omit `.spec.provider`
or set it to `generic`; mismatches are rejected.

### Prefer object-level identity where available

The `ObjectLevelWorkloadIdentity` gate supports per-object and per-tenant
identities. Applicable paths include Kustomization SOPS KMS access, OCI and
image registries, buckets, Azure DevOps Git repositories, notification
providers, and remote EKS, AKS, or GKE reconciliation. Newer source flows also
support AWS CodeCommit and Vault-compatible services through Kubernetes
Workload Identity. Check the exact fields in
[sources-auth-and-artifacts.md](references/sources-auth-and-artifacts.md).

### Verify and sign with the intended trust model

OCI artifact and container image verification supports Cosign v3. Source
verification can use a private Sigstore trusted root for self-hosted Rekor and
Fulcio. GitRepository verification accepts SSH-signed commits, while image
automation and bootstrap can create SSH-signed commits.

## Artifact and image workflows

### Compose or split artifacts

Enable the optional source-watcher during bootstrap or installation with
`--components-extra=source-watcher`. `ArtifactGenerator` can combine
GitRepository, OCIRepository, and Bucket content into `ExternalArtifact`
objects, split monorepos by path-specific copy globs, and process Helm charts.
`spec.pathPattern` can discover directories and use named captures in generated
artifact names, labels, and copy rules.

### Pin image tags to observed digests

Set `ImagePolicy.spec.digestReflectionPolicy: Always`. Image automation can
then write `<registry>/<name>:<tag>@<digest>`. For resources that store image
parts separately, use the `:name`, `:tag`, and `:digest` policy markers.

## Events, observability, and receivers

### Carry deployment identity through events

Annotations on Kustomizations and HelmReleases can add notification metadata.
Use `event.toolkit.fluxcd.io/image` for the full updated image,
`event.toolkit.fluxcd.io/change_request` for pull or merge request comments,
and `event.toolkit.fluxcd.io/commit` for commit statuses.

Git status providers can compute a monorepo-safe identifier with
`Provider.spec.commitStatusExpr`. An `otel` Provider turns source events into
root spans and consuming Kustomization or HelmRelease events into child spans.

### Secure and target Receiver triggers

A Receiver can filter its declared resources with CEL. Generic Receivers can
validate an OIDC ID token in place of an HMAC shared secret, and can be invoked
with:

```bash
flux trigger receiver
```

## Debugging cautions

Inspect effective merged inputs with:

```bash
flux debug kustomization --show-vars
flux debug helmrelease --show-values
```

These commands print values read from referenced Secrets in clear text. Treat
terminal output, logs, captures, and copied diagnostics as sensitive.

For HelmRelease inventory questions, inspect `.status.inventory`. For UI-based
rollout, workload, and multi-pod log inspection, the Flux Operator Web UI uses
Kubernetes RBAC and user impersonation; grant only the actions and log access
the operator should have.
