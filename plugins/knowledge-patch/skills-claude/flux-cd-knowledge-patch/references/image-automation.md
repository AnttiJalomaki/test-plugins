# Image automation

## Stable v1 APIs and migration

Since 2.7.0, these resources are stable at
`image.toolkit.fluxcd.io/v1`:

- `ImageRepository`
- `ImagePolicy`
- `ImageUpdateAutomation`

The image-reflector-controller `autologin` flags have been removed. Set
`ImageRepository.spec.provider` for cloud registries instead. The provider
must match the repository URL; use `generic` or omit the field for public
repositories and image-pull-secret authentication.

Commit templates must replace removed `.Updated` and
`.Changed.ImageResult` fields with `.Changed.FileChanges`,
`.Changed.Objects`, or the flat `.Changed.Changes` list.

`ImagePolicy.spec.suspend` can pause policy evaluation.

## Digest pinning

Since 2.6.0, set `ImagePolicy.spec.digestReflectionPolicy` to `Always` to
track the current digest. ImageUpdateAutomation can then write references in
the form `<registry>/<name>:<tag>@<digest>`.

The `:digest` image-policy marker supports custom resources that store
repository, tag, and digest separately:

```yaml
spec:
  values:
    image:
      repository: docker.io/my-org/my-app # {"$imagepolicy": "flux-system:my-app:name"}
      tag: latest # {"$imagepolicy": "flux-system:my-app:tag"}
      digest: sha256:ec0119... # {"$imagepolicy": "flux-system:my-app:digest"}
```

An image-policy marker can also update
`event.toolkit.fluxcd.io/image` on a workload alongside the actual image
value. Notification providers then receive the new full image reference in
the event body (since 2.5.0):

```yaml
metadata:
  annotations:
    event.toolkit.fluxcd.io/image: docker.io/org/my-app:1.0.0 # {"$imagepolicy": "apps:my-app"}
spec:
  values:
    image:
      tag: 1.0.0 # {"$imagepolicy": "apps:my-app:tag"}
```

## Git checkout and authentication

Since 2.7.0, image-automation-controller can enable Git sparse checkout with:

```text
--feature-gates=GitSparseCheckout=true
```

It can use Kubernetes Workload Identity with Azure DevOps repositories.

ImageUpdateAutomation also supports GitHub App authentication through
`.spec.secretRef.name`. Create the Secret with `flux create secret githubapp`;
newer flows can discover the GitHub App installation ID from the repository
owner.

## Commit signing

Since 2.9.0, image automation can sign pushed commits with an SSH key through
`ImageUpdateAutomation.spec.git.commit.signingKey`.

`flux bootstrap` can also SSH-sign the manifest commits it creates.

