---
name: flux-cd-knowledge-patch
description: Flux CD
version: 2.9.0
license: MIT
metadata:
  author: Nevaberry
---


# Flux CD Knowledge Patch

Use this skill when designing, upgrading, operating, or debugging Flux
installations and custom resources. Inspect the installed Flux and Kubernetes
versions first, then apply only guidance relevant to that installation.

## Reference index

| Reference | Topics |
| --- | --- |
| [upgrades-and-platforms.md](references/upgrades-and-platforms.md) | Required API migrations, breaking defaults, support windows |
| [kustomizations.md](references/kustomizations.md) | CEL health, deletion, SOPS, dependencies, reconciliation and apply controls |
| [helm-releases.md](references/helm-releases.md) | Helm v4 behavior, retries, values, inventory, chart processing |
| [sources-and-artifacts.md](references/sources-and-artifacts.md) | Git and OCI sources, artifact commands, ArtifactGenerator, verification |
| [image-automation.md](references/image-automation.md) | Stable image APIs, digest pinning, commit templates and signing |
| [notifications-and-receivers.md](references/notifications-and-receivers.md) | Event metadata, statuses, providers, traces, Receiver security |
| [identity-and-operations.md](references/identity-and-operations.md) | Workload Identity, GitHub Apps, CLI plugins, debugging, Operator UI |

## Upgrade blockers first

### Migrate stored APIs before replacing CRDs

Run the migration against the live cluster before an upgrade that removes
served beta versions:

```shell
flux migrate
```

Do not treat a manifest-only API rewrite as sufficient. Objects already stored
in Kubernetes etcd must also be migrated.

Removed versions accumulate across upgrades:

| Upgrade target | Versions removed by that target |
| --- | --- |
| 2.7 | `source` v1beta1, `kustomize` v1beta1, `helm` v2beta1, `image` v1beta1, `notification` v1beta1 |
| 2.8 | `source` v1beta2, `kustomize` v1beta2, `helm` v2beta2 |
| 2.9 | `image` v1beta2, `notification` v1beta2 |

For 2.9, also add `email` and `audience` to the Secret referenced by every GCR
Receiver.

### Preserve the old Helm post-render behavior deliberately

The HelmRelease post-render default is now `combined`; Helm hooks pass through
post-rendering. Before upgrading, set the strategy to `nohooks` explicitly for
charts that rely on hooks bypassing post-renderers.

### Account for Helm v4 defaults

New releases use server-side apply. Existing Helm-stored releases stay on
client-side apply until explicitly opted in. Kstatus health checking is the
default for every HelmRelease, and CEL can define health for Helm-managed
objects. Enable `UseHelm3Defaults` when the previous apply and health behavior
must be retained.

### Update image automation before enabling v1

Use `image.toolkit.fluxcd.io/v1` for `ImageRepository`, `ImagePolicy`, and
`ImageUpdateAutomation`. Replace removed image-reflector `autologin` flags with
`ImageRepository.spec.provider`.

Rewrite commit templates that use `.Updated` or `.Changed.ImageResult` to use:

- `.Changed.FileChanges`
- `.Changed.Objects`
- the flat `.Changed.Changes` list

### Validate registry providers strictly

For `OCIRepository` and `ImageRepository`, set `.spec.provider` to `aws`,
`azure`, or `gcp` only when the repository URL matches that cloud registry and
automatic OIDC authentication is intended. For public repositories or
image-pull-secret authentication, omit it or use `generic`.

## High-value reconciliation controls

### React to referenced configuration changes

Opt an individual ConfigMap or Secret into immediate reconciliation:

```yaml
metadata:
  labels:
    reconcile.fluxcd.io/watch: Enabled
```

The watched references include Kustomization `postBuild.substituteFrom`,
`decryption.secretRef`, and both kubeConfig references; HelmRelease
`valuesFrom` and both kubeConfig references; and Receiver `secretRef`.

Controllers can instead watch matching objects globally:

```text
--watch-configs-label-selector=owner!=helm
```

### Define health and dependency readiness with CEL

Use `Kustomization.spec.healthCheckExprs` for custom resources whose readiness
does not follow Kubernetes conventions:

```yaml
spec:
  wait: true
  healthCheckExprs:
    - apiVersion: cluster.x-k8s.io/v1beta1
      kind: Cluster
      failed: "status.conditions.filter(e, e.type == 'Ready').all(e, e.status == 'False')"
      current: "status.conditions.filter(e, e.type == 'Ready').all(e, e.status == 'True')"
```

CEL expressions are also available on Kustomization and HelmRelease
`dependsOn` entries. Health expressions may leave `kind` empty to match every
resource kind in an API group.

### Keep externally owned fields out of apply

Use `Kustomization.spec.ignore` to exclude selected managed fields from drift
detection and apply:

```yaml
spec:
  ignore:
    - target:
        kind: Deployment
      paths:
        - /spec/replicas
```

This is appropriate for fields such as replicas owned by an HPA.

### Choose deletion completion semantics

`Kustomization.spec.deletionPolicy` controls garbage collection.
`WaitForTermination` waits for all managed resources to disappear before the
Kustomization itself can finish deletion.

## Common source and delivery patterns

### Authenticate as a GitHub App

Create the credential Secret, then reference it from `GitRepository` or
`ImageUpdateAutomation` through `.spec.secretRef.name`:

```shell
flux create secret githubapp github-auth \
  --app-id=1 \
  --app-installation-id=2 \
  --app-private-key=~/private-key.pem
```

`flux create source git --provider=github` supports this mode. Newer flows can
discover the installation ID from the repository owner, and GitRepository
authentication can be combined with mTLS.

### Compose or split artifacts

Install the optional source-watcher component with:

```text
--components-extra=source-watcher
```

`ArtifactGenerator` can combine `GitRepository`, `OCIRepository`, and `Bucket`
content, process Helm charts, or split a monorepo into `ExternalArtifact`
objects. Consumers can use `sourceRef.kind: ExternalArtifact` or
`HelmRelease.spec.chartRef.kind: ExternalArtifact`. `spec.pathPattern` adds
directory discovery with named captures available to artifact templates.

### Pin image digests while retaining tags

Set `ImagePolicy.spec.digestReflectionPolicy: Always`.
ImageUpdateAutomation can then write `<name>:<tag>@<digest>`. Use the
`:digest` marker when repository, tag, and digest live in separate fields:

```yaml
image:
  repository: docker.io/my-org/my-app # {"$imagepolicy": "flux-system:my-app:name"}
  tag: latest # {"$imagepolicy": "flux-system:my-app:tag"}
  digest: sha256:ec0119... # {"$imagepolicy": "flux-system:my-app:digest"}
```

## Helm failure handling

Use the HelmRelease `RetryOnFailure` install or upgrade strategy for failed
releases. When enabling `CancelHealthCheckOnNewRevision` on helm-controller,
also enable `DefaultToRetryOnFailure`; otherwise cancellation under the
default no-retry configuration can leave a release stuck.

Cancellation applies to new source revisions, spec changes, watched reference
changes, manual reconciliations, and Receiver triggers. It reports
`HealthCheckCanceled` on the `Ready` condition.

For OCI charts, `DisableChartDigestTracking` disables the default behavior of
appending the chart digest to the chart version.

## Notification and observability patterns

- Add `event.toolkit.fluxcd.io/*` annotations to Kustomizations and
  HelmReleases to enrich event metadata.
- Use `Provider.spec.commitStatusExpr` to derive distinct commit status
  identifiers for clusters or workloads in a monorepo.
- A Provider of type `otel` emits source reconciliations as root spans and
  consuming Kustomization or HelmRelease reconciliations as child spans.
- Pull or merge request comment providers consume
  `event.toolkit.fluxcd.io/change_request`; status providers consume
  `event.toolkit.fluxcd.io/commit`.
- Generic Receivers can validate an OIDC ID token, and
  `flux trigger receiver` invokes them without manually building a webhook
  request.

## Sensitive debugging output

These commands show merged data from inline configuration, ConfigMaps, and
Secrets:

```shell
flux debug kustomization --show-vars
flux debug helmrelease --show-values
```

Referenced Secret values are printed in clear text. Treat command output,
terminal scrollback, captured logs, and CI artifacts as sensitive.

