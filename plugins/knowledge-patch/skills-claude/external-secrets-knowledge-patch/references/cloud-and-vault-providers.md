# Cloud, Vault, and Kubernetes Providers

This reference covers high-use cloud providers plus Vault, OpenBao, Kubernetes,
IBM, Yandex, and Volcengine. Write-specific behavior is consolidated in
`push-secrets.md`.

## AWS

### Secrets Manager

- `AWSProvider.prefix` applies a common remote-secret name prefix at the provider
  level (0.16.0).
- When a Secrets Manager `find` filter supplies both a name and tags, both
  criteria are applied (1.2.0).
- With explicit AWS credentials, resolution does not fall back to EC2 Instance
  Metadata Service (2.2.0). A configuration error therefore cannot silently
  change identity to the instance role.
- Kubernetes context is injected into STS sessions as session tags (2.5.0).
- AWS Certificate Manager is available as a provider (2.8.0).

For tags, resource policies, metadata-only reconciliation, empty policy values,
and replication locations in a write workflow, see `push-secrets.md`.

### Parameter Store

The provider can set the Parameter Store tier (0.13.0). Make tier selection
explicit where cost, size, throughput, or policy depends on it.

### ECR credentials

The `ECRAuthorizationToken` generator accepts custom ECR endpoints (0.18.0).
Credential resolution through the normal AWS credential chain was restored in
0.19.0. Check both endpoint and identity chain when token generation fails.

## Google Cloud

### Authentication and project selection

- Workload Identity parameters are optional when they do not apply; placeholder
  values are unnecessary (0.16.0).
- The provider supports Workload Identity Federation (0.20.0).
- GCP Secret Manager can auto-detect `projectID` from the metadata server
  (2.2.0).
- Workload Identity Federation through a Kubernetes service account supports
  service-account impersonation (2.3.0).
- The service-account email is optional for that WIF impersonation setup
  (2.5.0).

### Secret behavior

Azure-style assumptions do not carry over to GCP. Location affects whether the
provider searches regional or global secrets, and regional push behavior has
special replication rules. See `push-secrets.md` for the location, existence,
version, and replication semantics added across 0.17.0, 0.18.0, 1.1.0, 1.2.0,
and 2.4.0.

## HashiCorp Vault

### Dynamic secrets and request behavior

- `allowEmptyResponse` allows a Vault dynamic-secret request to accept an empty
  response when that is intentional (0.13.0).
- `VaultDynamicSecret` GET requests can take parameters from the resource spec;
  GET uses its own parameter set (2.4.0).
- Vault v2 custom metadata exposes additional values for use by ESO (2.8.0).

### Authentication and client caching

- Vault clients are cached per namespace where required, preventing one
  namespace-specific client from being reused for another namespace (0.17.0).
- Vault authentication supports Pod Identity (0.20.0).
- Vault authentication also supports GCP Workload Identity (1.1.0).
- Token caching is graduated from experimental, and token expiry participates in
  cache validation (2.3.0).
- TLS authentication accepts a `VaultRole` attribute (2.3.0).

The provider implements the existence and set operations required for push as
of 0.20.0; consult `push-secrets.md` for lifecycle implications.

## OpenBao

OpenBao initially worked through the Vault-compatible provider (0.17.0). A
dedicated OpenBao provider is available as of 2.7.0 and supports:

- custom trust through `caBundle` or `caProvider`;
- `auth.userPass` and `auth.appRole`;
- OpenBao namespaces.

Prefer the dedicated provider for new configurations and validate migrations
from a Vault-shaped provider block against the installed CRD.

## Kubernetes provider

- The `auth` field is optional; omit it when the configured access path does not
  need an explicit authentication block (0.20.0).
- Fetched-secret metadata can include the remote namespace (0.20.0).
- If no CA is configured, the provider falls back to system CA roots (2.1.0).
- ConfigMap access through `CAProvider` works correctly (2.4.0).

TokenRequest namespace handling and least-privilege RBAC are documented in
`helm-and-operations.md`.

Whole-Secret deletion, existence checks, and replace-not-merge push semantics
are documented in `push-secrets.md`.

## Azure Key Vault

- Fetched Azure secrets expose their expiration time (2.2.0).
- Azure `PushSecret` mappings can set `contentType` (2.4.0).

## IBM Cloud Secrets Manager

- The provider accepts custom credentials (0.18.0).
- API-key authentication can override the IBM IAM endpoint (1.1.0), enabling a
  non-default identity endpoint.

## Yandex

Yandex Lockbox and Certificate Manager can retrieve secrets and certificates by
name rather than requiring only other identifiers (0.20.0).

## Volcengine

Volcengine is available as a provider (0.20.0). On a `ClusterSecretStore`, it
honors `secretRef.namespace`, so the credential Secret is resolved from the
configured namespace (2.7.0).
