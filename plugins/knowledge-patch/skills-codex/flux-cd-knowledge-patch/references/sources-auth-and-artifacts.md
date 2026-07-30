# Sources, authentication, and artifacts

## Contents

- [GitHub App authentication](#github-app-authentication)
- [GitRepository transport and checkout](#gitrepository-transport-and-checkout)
- [OCI repositories and registry authentication](#oci-repositories-and-registry-authentication)
- [Workload Identity coverage](#workload-identity-coverage)
- [Commit signing and source verification](#commit-signing-and-source-verification)
- [ArtifactGenerator and ExternalArtifact](#artifactgenerator-and-externalartifact)

## GitHub App authentication

Since 2.5.0, source-controller and image-automation-controller can authenticate
to GitHub repositories as a GitHub App installation. Create the Secret and
reference it through `.spec.secretRef.name` on a `GitRepository` or
`ImageUpdateAutomation`:

```bash
flux create secret githubapp github-auth \
  --app-id=1 \
  --app-installation-id=2 \
  --app-private-key=~/private-key.pem
```

`flux create source git --provider=github` supports the same mode. GitRepository
GitHub App authentication also supports mutual TLS (since 2.7.0). Since 2.8.0,
Flux can look up the installation ID from the repository owner, so supported
flows do not require it to be supplied manually.

## GitRepository transport and checkout

Since 2.6.0, `GitRepository` v1 accepts a directory list in
`.spec.sparseCheckout` and HTTPS Git repositories can authenticate with mutual
TLS:

```yaml
spec:
  sparseCheckout:
    - apps
    - clusters/production
```

## OCI repositories and registry authentication

`OCIRepository` is GA at `source.toolkit.fluxcd.io/v1` (since 2.6.0) and is
backward compatible with `v1beta2`; migration requires only an `apiVersion`
change.

`OCIRepository` and `ImageRepository` reject a `.spec.provider` that does not
match the repository URL. Use `aws`, `azure`, or `gcp` only for a matching
registry with automatic OIDC authentication. For public repositories or
image-pull-secret authentication, omit the provider or set it to `generic`.

The opt-in `ObjectLevelWorkloadIdentity` gate supports per-object and
per-tenant identities for `OCIRepository` and `ImageRepository` registry
access (since 2.6.0).

## Workload Identity coverage

The object-level Kubernetes Workload Identity integrations added in 2.7.0
include:

- `Bucket.spec.serviceAccountName` for S3, Azure Blob Storage, and GCS.
- `GitRepository.spec.serviceAccountName` for Azure DevOps.
- `Provider.spec.serviceAccountName` for Azure DevOps, Azure Event Hub, and
  Google Pub/Sub.
- `Kustomization.spec.kubeConfig.configMapRef.name` and
  `HelmRelease.spec.kubeConfig.configMapRef.name` for remote EKS, AKS, and GKE
  authentication without static kubeconfig Secrets.

Since 2.7.0, image-automation-controller can also use Kubernetes Workload
Identity for Azure DevOps repositories. Since 2.9.0, GitRepository access and
bootstrap support AWS CodeCommit through Workload Identity.

For SOPS and Vault-compatible decryption identity, see
[kustomizations-and-helm.md](kustomizations-and-helm.md).

## Commit signing and source verification

Since 2.9.0:

- `GitRepository.spec.verify` accepts SSH-signed Git commits as well as GPG
  signatures.
- `ImageUpdateAutomation.spec.git.commit.signingKey` can SSH-sign pushed
  commits.
- `flux bootstrap` can SSH-sign its manifest commits.
- Source-controller can use a custom Sigstore trusted root for keyless OCI
  artifact and container image verification. This permits air-gapped systems
  to use self-hosted Rekor and Fulcio infrastructure.

OCI artifact and container image verification supports Cosign v3 (since
2.8.0).

## ArtifactGenerator and ExternalArtifact

Install the optional source-watcher component with
`--components-extra=source-watcher` during bootstrap or installation (since
2.7.0). Its `ArtifactGenerator` can combine GitRepository, OCIRepository, and
Bucket content or split a monorepo into independently revised
`ExternalArtifact` objects.

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

A HelmRelease can consume the result with `chartRef.kind: ExternalArtifact`.
Path-specific `copy.from` globs can produce multiple artifacts from a monorepo;
a Kustomization uses `sourceRef.kind: ExternalArtifact`, so only the artifact
whose paths changed triggers that deployment.

ArtifactGenerator can extract and modify Helm charts (since 2.8.0).

Since 2.9.0, `ArtifactGenerator.spec.pathPattern` discovers matching monorepo
directories. Named captures become template variables for artifact names,
labels, and copy rules:

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
