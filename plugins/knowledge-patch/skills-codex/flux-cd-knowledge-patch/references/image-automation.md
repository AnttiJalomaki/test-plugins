# Image automation

## Stable APIs and migration

Since 2.7.0, `ImageRepository`, `ImagePolicy`, and `ImageUpdateAutomation` are
stable at `image.toolkit.fluxcd.io/v1`.

The image-reflector-controller `autologin` flags have been removed. Select
cloud-registry login with `ImageRepository.spec.provider` instead. A provider
must match the repository URL; use `aws`, `azure`, or `gcp` only for the
matching registry and automatic OIDC authentication. Omit it or use `generic`
for public repositories and image-pull-secret authentication.

Commit templates must replace removed `.Updated` and `.Changed.ImageResult`
fields with `.Changed.FileChanges`, `.Changed.Objects`, or the flat
`.Changed.Changes` list.

## Digest pinning and markers

Set `ImagePolicy.spec.digestReflectionPolicy` to `Always` to track the newest
digest (since 2.6.0). `ImageUpdateAutomation` can then write
`<registry>/<name>:<tag>@<digest>`.

For custom resources that keep the parts separate, use image-policy markers
for name, tag, and digest:

```yaml
spec:
  values:
    image:
      repository: docker.io/my-org/my-app # {"$imagepolicy": "flux-system:my-app:name"}
      tag: latest # {"$imagepolicy": "flux-system:my-app:tag"}
      digest: sha256:ec0119... # {"$imagepolicy": "flux-system:my-app:digest"}
```

An image-policy marker can also update an
`event.toolkit.fluxcd.io/image` annotation alongside a workload value (since
2.5.0). Providers then receive the new full image reference in the message:

```yaml
metadata:
  annotations:
    event.toolkit.fluxcd.io/image: docker.io/org/my-app:1.0.0 # {"$imagepolicy": "apps:my-app"}
spec:
  values:
    image:
      tag: 1.0.0 # {"$imagepolicy": "apps:my-app:tag"}
```

## Policy and checkout controls

`ImagePolicy.spec.suspend` pauses policy evaluation (since 2.7.0).

Enable sparse checkout for `ImageUpdateAutomation` on
image-automation-controller with `--feature-gates=GitSparseCheckout=true`
(since 2.7.0).

## Authentication and signing

GitHub App installation authentication is available for
image-automation-controller (since 2.5.0). Reference the generated Secret with
`.spec.secretRef.name` on `ImageUpdateAutomation`.

The opt-in `ObjectLevelWorkloadIdentity` gate permits per-object registry
identity for ImageRepository (since 2.6.0). Image automation can use Kubernetes
Workload Identity for Azure DevOps repositories (since 2.7.0).

Since 2.9.0, `ImageUpdateAutomation.spec.git.commit.signingKey` can sign pushed
commits with an SSH key.
