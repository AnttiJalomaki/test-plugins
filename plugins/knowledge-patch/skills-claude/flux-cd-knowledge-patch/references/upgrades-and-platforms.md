# Upgrades and supported platforms

## Required stored-API migrations

Before replacing CRDs, run `flux migrate` against the live cluster. This
updates resources stored in Kubernetes etcd; merely changing checked-in
manifests does not migrate stored objects.

### Upgrade to 2.7

Flux 2.7.0 CRDs remove these API versions:

- `source.toolkit.fluxcd.io/v1beta1`
- `kustomize.toolkit.fluxcd.io/v1beta1`
- `helm.toolkit.fluxcd.io/v2beta1`
- `image.toolkit.fluxcd.io/v1beta1`
- `notification.toolkit.fluxcd.io/v1beta1`

Run `flux migrate` before the upgrade.

### Upgrade to 2.8

Flux 2.8.0 CRDs remove:

- `source.toolkit.fluxcd.io/v1beta2`
- `kustomize.toolkit.fluxcd.io/v1beta2`
- `helm.toolkit.fluxcd.io/v2beta2`

Migrate stored resources to stable APIs before upgrading:

```shell
flux migrate
```

### Upgrade to 2.9

Flux 2.9.0 CRDs remove:

- `image.toolkit.fluxcd.io/v1beta2`
- `notification.toolkit.fluxcd.io/v1beta2`

Run `flux migrate` before the upgrade. GCR Receivers also require `email` and
`audience` in their referenced Secret.

## Breaking and compatibility-sensitive defaults

### Helm v4 apply and health

Since 2.8.0, Flux ships Helm v4:

- New releases use server-side apply.
- Releases already stored by Helm continue using client-side apply until
  explicitly opted in.
- Kstatus-based health checking is the default for every HelmRelease.
- CEL expressions can define readiness for Helm-managed objects.

Enable the `UseHelm3Defaults` feature gate to retain the prior apply and health
behavior.

### Helm post-rendering

Since 2.9.0, the HelmRelease post-render strategy defaults to `combined`
instead of `nohooks`. Helm hooks therefore pass through post-rendering. Set
`nohooks` explicitly before upgrading when a chart relies on the old behavior.

### Image automation v1

Since 2.7.0, `ImageRepository`, `ImagePolicy`, and `ImageUpdateAutomation` are
stable at `image.toolkit.fluxcd.io/v1`.

The image-reflector-controller `autologin` flags have been removed. Configure
cloud registry login with `ImageRepository.spec.provider`. Commit templates
must replace the removed `.Updated` and `.Changed.ImageResult` fields with
`.Changed.FileChanges`, `.Changed.Objects`, or the flat `.Changed.Changes`
list.

## Platform support window

Flux supports only the latest three Kubernetes minor versions. Check both the
Flux and cluster versions before installing or upgrading.

| Flux batch | Kubernetes | OpenShift | Newly end-of-life Flux line |
| --- | --- | --- | --- |
| 2.5.0 | 1.30–1.32 | 4.17 | 2.2 |
| 2.6.0 | 1.31–1.33 | 4.18 | 2.3 |
| 2.7.0 | 1.32–1.34 | 4.19 | 2.4 |
| 2.8.0 | 1.33–1.35 | 4.20 | 2.5 |
| 2.9.0 | 1.34–1.36 | 4.21 | 2.6 |

