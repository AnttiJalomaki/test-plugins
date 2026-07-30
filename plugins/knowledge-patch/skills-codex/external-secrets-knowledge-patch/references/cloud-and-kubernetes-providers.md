# Cloud and Kubernetes providers

## AWS

### Secrets Manager

- Set `AWSProvider.prefix` to apply a common secret-name prefix at provider
  level (since 0.16.0).
- AWS tags are supported, including update, patch, and delete lifecycle
  operations (since 0.16.0 and 0.19.0).
- When a find filter supplies both name and tags, both criteria are applied
  (since 1.2.0).
- Tags and resource policies synchronize even if the secret value has not
  changed (since 2.2.0).
- Resource policies are converted to canonical, sorted JSON before comparison
  so ordering alone does not create a difference (since 1.2.0).
- `replicationLocations` configures secret replicas (since 2.7.0).

### Parameter Store

The integration can configure the parameter tier (since 0.13.0).

### Authentication and generated credentials

- `ECRAuthorizationToken` can target a custom ECR endpoint (since 0.18.0) and
  resolves credentials through the AWS credential chain (since 0.19.0).
- Explicit credentials no longer fall back to EC2 Instance Metadata Service
  (since 2.2.0).
- Kubernetes context is injected into STS sessions as session tags (since
  2.5.0).
- The `STSSessionToken` generator no longer supports JWT-token authentication;
  choose another supported path (since 0.19.0).

### Certificate Manager

AWS Certificate Manager is available as a provider (since 2.8.0).

## GCP Secret Manager

- Workload Identity parameters are optional when not applicable (since
  0.16.0).
- Workload Identity Federation is supported (since 0.20.0).
- Workload Identity Federation through a Kubernetes ServiceAccount supports
  service-account impersonation (since 2.3.0).
- A service-account email is optional for WIF impersonation (since 2.5.0).
- `projectID` can be auto-detected from the GCP metadata server (since 2.2.0).
- When a store location is configured, existence checks search for regional
  secrets (since 1.2.0).
- Pushes apply location and replication settings (since 0.17.0); regional push
  operations omit replication settings (since 0.18.0), and multiple
  `replicationLocations` are supported (since 2.4.0).
- Push handling checks whether a secret version exists before treating the
  target as usable (since 1.1.0).

## Azure Key Vault

- Fetched secrets include their expiration time (since 2.2.0).
- `PushSecret` supports `contentType` (since 2.4.0).

## IBM Cloud Secrets Manager

- Custom credentials are supported (since 0.18.0).
- API-key authentication can override the IAM endpoint (since 1.1.0).

## Kubernetes provider

- The `auth` block is optional (since 0.20.0).
- Fetched secret metadata can include the remote namespace (since 0.20.0).
- The provider falls back to system CA roots when no CA is configured (since
  2.1.0).
- It implements `SecretExists` (since 2.1.0).
- It can delete an entire Secret instead of deleting keys one at a time (since
  1.1.0).
- Push operations replace the complete remote Secret; keys absent from the
  pushed Secret do not remain (since 2.7.0).
- Provider TokenRequests use the URL namespace in the request body too,
  keeping requests consistent when the target namespace differs from the
  caller context (since 2.6.0).

## Cloud.ru Secret Manager

Cloud.ru is available as a provider (since 0.15.0) and supports provider paths
(since 2.2.0).

## Yandex

Yandex Lockbox and Certificate Manager can fetch secrets and certificates by
name, not only by their other identifiers (since 0.20.0).

## Volcengine

Volcengine is available as a provider (since 0.20.0). On a
`ClusterSecretStore`, it honors `secretRef.namespace` when resolving referenced
credentials (since 2.7.0).

## Barbican

Barbican is available as a provider (since 1.2.0). It supports `property` and
`extract`, and `find.name.regexp` is interpreted as a regular expression
rather than an exact name (since 2.8.0).

## OVHcloud

OVHcloud is available as a provider (since 2.3.0).

## Removed cloud providers

Alibaba and Device42 were removed because they were unsupported and
unmaintained. Migrate any dependent stores before upgrading to 2.0.0.
