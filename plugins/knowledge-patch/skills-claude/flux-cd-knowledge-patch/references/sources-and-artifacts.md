# Sources and artifacts

## OCIRepository and artifact commands

Since 2.6.0, `OCIRepository` is GA at
`source.toolkit.fluxcd.io/v1`. It is backward compatible with `v1beta2`, so
manifests can migrate by changing only `apiVersion`.

These artifact commands are stable:

```shell
flux build artifact
flux push artifact
flux pull artifact
flux tag artifact
flux diff artifact
flux list artifacts
```

The stable media types are:

- config: `application/vnd.cncf.flux.config.v1+json`
- content: `application/vnd.cncf.flux.content.v1.tar+gzip`

## Registry provider validation

Since 2.6.0, `OCIRepository` and `ImageRepository` reject a `.spec.provider`
that does not match the repository URL.

Use `aws`, `azure`, or `gcp` only with the corresponding cloud registry when
automatic OIDC authentication is intended. For public repositories or
image-pull-secret authentication, omit the provider or set it to `generic`.

The opt-in `ObjectLevelWorkloadIdentity` feature gate permits identity to be
assigned per object and tenant for OCIRepository and ImageRepository registry
access.

## GitRepository checkout and transport

Since 2.6.0, `GitRepository` v1 accepts directories in `.spec.sparseCheckout`
to fetch only selected paths:

```yaml
spec:
  sparseCheckout:
    - apps
    - clusters/production
```

HTTPS Git repositories also support mutual TLS. Since 2.7.0, GitHub App
authentication for GitRepository can be combined with mTLS.

## Git commit verification

Since 2.9.0, `GitRepository.spec.verify` accepts SSH-signed commits in addition
to GPG signatures.

AWS CodeCommit access and `flux bootstrap` also support AWS Workload Identity,
allowing keyless access to CodeCommit.

## OCI and image verification

Cosign v3 verification for OCI artifacts and container images is supported
since 2.8.0.

Since 2.9.0, source-controller can use a custom Sigstore trusted root for
keyless OCI artifact and container image verification. This supports
air-gapped installations with self-hosted Rekor and Fulcio infrastructure.

## ArtifactGenerator and ExternalArtifact

The optional source-watcher component was introduced in 2.7.0. Enable it at
bootstrap or installation:

```text
--components-extra=source-watcher
```

`ArtifactGenerator` can combine content from `GitRepository`,
`OCIRepository`, and `Bucket` sources, or split a monorepo into independently
revised `ExternalArtifact` objects:

```yaml
apiVersion: source.extensions.fluxcd.io/v1beta1
kind: ArtifactGenerator
metadata:
  name: podinfo
  namespace: apps
spec:
  sources:
    - alias: chart
      kind: OCIRepository
      name: podinfo-chart
    - alias: repo
      kind: GitRepository
      name: podinfo-values
  artifacts:
    - name: podinfo-composite
      originRevision: "@chart"
      copy:
        - from: "@chart/"
          to: "@artifact/"
        - from: "@repo/charts/podinfo/values.yaml"
          to: "@artifact/podinfo/values.yaml"
          strategy: Overwrite
        - from: "@repo/charts/podinfo/values-prod.yaml"
          to: "@artifact/podinfo/values.yaml"
          strategy: Merge
```

Kustomizations consume generated output with
`sourceRef.kind: ExternalArtifact`. A HelmRelease uses
`spec.chartRef.kind: ExternalArtifact`. With multiple artifact entries and
path-specific `copy.from` globs, only the artifact whose paths changed
triggers its deployment.

Since 2.8.0, ArtifactGenerator can extract and modify Helm charts while
producing output.

Since 2.9.0, `ArtifactGenerator.spec.pathPattern` discovers matching monorepo
directories. Named captures become variables for artifact names, labels, and
copy rules:

```yaml
spec:
  sources:
    - alias: monorepo
      kind: GitRepository
      name: my-monorepo
  pathPattern: "@monorepo/apps/{app}/envs/{env}"
  artifacts:
    - name: "{app}-{env}"
      copy:
        - from: "@monorepo/apps/{app}/envs/{env}/**"
          to: "@artifact/"
```

