# Identity and operations

## Object-level Workload Identity

The opt-in `ObjectLevelWorkloadIdentity` feature gate was introduced in
2.6.0. It permits identities to be assigned per object and tenant for:

- Kustomization SOPS decryption through KMS services;
- OCIRepository access to container registries;
- ImageRepository access to container registries.

The scope expanded in 2.7.0:

- `Bucket.spec.serviceAccountName` for S3, Azure Blob Storage, and GCS;
- `GitRepository.spec.serviceAccountName` for Azure DevOps;
- `Provider.spec.serviceAccountName` for Azure DevOps, Azure Event Hub, and
  Google Pub/Sub.

Remote EKS, AKS, and GKE authentication no longer requires static kubeconfig
Secrets. Configure:

- `Kustomization.spec.kubeConfig.configMapRef.name`, or
- `HelmRelease.spec.kubeConfig.configMapRef.name`.

Since 2.9.0, GitRepository access and bootstrap support AWS CodeCommit through
Workload Identity. Kustomize-controller can authenticate to OpenBao and
HashiCorp Vault by exchanging a Kubernetes ServiceAccount token instead of
storing a long-lived Vault token.

## GitHub App authentication

Since 2.5.0, source-controller and image-automation-controller can
authenticate to GitHub repositories as a GitHub App installation.

Generate the Secret:

```shell
flux create secret githubapp github-auth \
  --app-id=1 \
  --app-installation-id=2 \
  --app-private-key=~/private-key.pem
```

Reference it through `.spec.secretRef.name` in a `GitRepository` or
`ImageUpdateAutomation`. `flux create source git --provider=github` supports
the same authentication mode.

Since 2.6.0, `github` and `githubdispatch` notification Providers also support
GitHub App authentication. Since 2.7.0, GitRepository GitHub App
authentication supports mTLS.

Since 2.8.0, supported flows can discover the installation ID automatically
from the repository owner, so it need not always be supplied manually.

## Debug merged configuration

Since 2.5.0, these commands display effective Kustomization substitutions or
HelmRelease values after merging inline configuration with referenced
ConfigMaps and Secrets:

```shell
flux debug kustomization --show-vars
flux debug helmrelease --show-values
```

They print referenced Secret values in clear text. Treat the output, terminal
scrollback, command logs, and CI artifacts as sensitive.

## Flux CLI plugins

Since 2.9.0, the CLI installs independently versioned plugins under
`~/fluxcd/plugins` and exposes them as `flux <plugin>`.

```shell
flux plugin search
flux plugin install schema@0.5.0
flux plugin list
flux plugin update schema
flux plugin uninstall schema
```

The initial catalog includes:

- Mirror, for declarative registry mirroring;
- Schema, for JSON Schema and CEL validation.

Pin plugin versions or immutable digests in reproducible automation.

## Flux Operator Web UI

The Flux Operator Web UI added in 2.8.0 provides:

- cluster and GitOps-resource monitoring;
- rollout inspection and delivery graphs;
- RBAC-guarded actions;
- OIDC single sign-on integrated with Kubernetes RBAC for multi-tenant
  clusters.

Since 2.9.0, it also provides a workload dashboard for Deployments,
StatefulSets, DaemonSets, and CronJobs, plus a multi-pod, multi-container log
viewer. Workload actions and log access continue to use Kubernetes RBAC
through user impersonation.
