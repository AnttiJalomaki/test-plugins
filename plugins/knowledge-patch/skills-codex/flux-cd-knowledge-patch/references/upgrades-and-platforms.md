# Upgrades and platform support

## Required API migrations

Run `flux migrate` before an upgrade whose CRDs stop serving an API that is
still used to store resources in Kubernetes etcd.

### Flux 2.7 boundary

Flux 2.7.0 removes these APIs:

- `source.toolkit.fluxcd.io/v1beta1`
- `kustomize.toolkit.fluxcd.io/v1beta1`
- `helm.toolkit.fluxcd.io/v2beta1`
- `image.toolkit.fluxcd.io/v1beta1`
- `notification.toolkit.fluxcd.io/v1beta1`

Migrate stored resources to their latest API versions before upgrading.

### Flux 2.8 boundary

Flux 2.8.0 removes:

- `source.toolkit.fluxcd.io/v1beta2`
- `kustomize.toolkit.fluxcd.io/v1beta2`
- `helm.toolkit.fluxcd.io/v2beta2`

Run `flux migrate` before installing those CRDs.

### Flux 2.9 boundary

Flux 2.9.0 removes:

- `image.toolkit.fluxcd.io/v1beta2`
- `notification.toolkit.fluxcd.io/v1beta2`

Run `flux migrate` before upgrading. GCR Receivers must also provide `email`
and `audience` in the referenced Secret.

## Defaults and behavior changes

### Helm v4 apply and health

Since 2.8.0, Flux ships Helm v4. New releases use server-side apply. Releases
already stored by Helm remain on client-side apply until explicitly opted in.
Kstatus-based health checking is now the default for every HelmRelease, and CEL
expressions can describe readiness for Helm-managed objects. Enable the
`UseHelm3Defaults` feature gate to retain the previous apply and health
behavior.

### Helm post-rendering

Since 2.9.0, the default HelmRelease post-render strategy is `combined` rather
than `nohooks`, so Helm hooks pass through post-rendering. Before upgrading,
set the strategy explicitly to `nohooks` if a chart depends on hooks bypassing
post-rendering.

### Image automation

Since 2.7.0, `ImageRepository`, `ImagePolicy`, and `ImageUpdateAutomation` are
stable at `image.toolkit.fluxcd.io/v1`. The image-reflector-controller
`autologin` flags are removed; use `ImageRepository.spec.provider` for cloud
registries. Commit templates must replace `.Updated` and
`.Changed.ImageResult` with `.Changed.FileChanges`, `.Changed.Objects`, or the
flat `.Changed.Changes` list.

### OCIRepository API

Since 2.6.0, `OCIRepository` is GA at
`source.toolkit.fluxcd.io/v1`. It is backward compatible with `v1beta2`, so a
manifest can migrate by changing only `apiVersion`.

## Platform compatibility

Flux supports the latest three Kubernetes minor releases. Match the Flux
release to the cluster platform:

| Flux release | Kubernetes | OpenShift | Release reaching end of life |
| --- | --- | --- | --- |
| 2.5.0 | 1.30–1.32 | 4.17 | Flux 2.2 |
| 2.6.0 | 1.31–1.33 | 4.18 | Flux 2.3 |
| 2.7.0 | 1.32–1.34 | 4.19 | Flux 2.4 |
| 2.8.0 | 1.33–1.35 | 4.20 | Flux 2.5 |
| 2.9.0 | 1.34–1.36 | 4.21 | Flux 2.6 |

Do not treat a newer Kubernetes minor as implicitly supported merely because
the controllers start successfully.
