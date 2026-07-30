# Vault and integration providers

## HashiCorp Vault

- Dynamic secrets accept `allowEmptyResponse` when an empty response is valid
  (since 0.13.0).
- Clients are cached separately by namespace where necessary, preventing
  namespace-specific access from sharing one cached client (since 0.17.0).
- The provider supports the existence and set operations required by
  `PushSecret` (since 0.20.0).
- Pod Identity is available as an authentication path (since 0.20.0).
- Vault authentication supports GCP Workload Identity (since 1.1.0).
- Token caching is no longer experimental, and token expiry participates in
  cache validation (since 2.3.0).
- TLS authentication accepts a `VaultRole` attribute (since 2.3.0).
- `VaultDynamicSecret` GET requests can take parameters from the resource spec;
  GET uses its own parameter set (since 2.4.0).
- Vault v2 custom metadata exposes additional values to External Secrets
  (since 2.8.0).

## OpenBao

OpenBao can be used through Vault compatibility (since 0.17.0). Prefer the
dedicated OpenBao provider where available (since 2.7.0); it supports custom
trust through `caBundle` or `caProvider`, `auth.userPass`, `auth.appRole`, and
OpenBao namespaces.

## 1Password

- An SDK-based provider is available (since 0.17.0).
- Connect is classified as read-write (since 0.18.0).
- The SDK provider implements `GetSecretMap` at parity with Connect and can
  select a vault by UUID (since 0.18.0).
- Authorization errors are retried so transient failures can recover (since
  0.20.0).
- Native item IDs are supported (since 2.2.0).
- The SDK provider supports multi-field pushes and completes its push
  implementation (since 2.3.0).
- Each client gets a fresh provider instance, preventing the race that could
  route an operation to the wrong vault (since 2.5.0).
- The SDK provider implements `GetAllSecrets` for bulk selection (since
  2.7.0).
- `PushSecret` honors `IfNotExists` (since 2.8.0).

## Infisical

- Missing secrets, bad authentication, and other failures that once failed
  silently are reported (since 0.13.0).
- `data` references can address secrets within paths (since 0.17.0).
- Authentication methods are configurable (since 0.19.0).
- Kubernetes authentication can use a Client JWT as its Reviewer JWT token
  (since 0.20.0).
- `dataFrom.find.path` filters by secret path, not by secret name (since
  2.2.0).
- Secret scopes can specify an organization slug (since 2.7.0).
- `PushSecret` is supported, and an HTTP 404 maps to `NoSecretErr` so missing
  data follows absence semantics (since 2.7.0).

## BeyondTrust

- Select the provider API version with its API-version parameter (since
  0.14.0).
- Get-secret operations accept `decrypt` to request decryption (since 1.3.0).
- Secret creation supports API v3.2 (since 2.5.0).
- BeyondTrust WorkloadCredentials is available as a provider (since 2.7.0).

## Delinea Secret Server

- Fetched secrets need not contain JSON (since 0.18.0).
- Connections support a domain field and lookup by path (since 0.20.0).
- TLS is configured for Secret Server connections (since 1.1.0).
- `PushSecret` is supported (since 2.3.0).
- Access-token authentication is supported (since 2.8.0).

## Passbolt

- The provider honors `refreshInterval` correctly (since 0.14.0).
- Passbolt V5 API is supported (since 2.2.0).
- Custom CA trust can use a CA bundle or CA provider (since 2.4.0).
- `ExternalSecret` supports the `v5-custom-fields` resource type (since
  2.6.0).

## Akeyless

- The provider is classified read-write, enabling push routing (since 2.7.0).
- `azure_ad` Workload Identity authentication uses `serviceAccountRef` (since
  2.8.0).
- Set `SecretStore.ignoreCache` to bypass the Akeyless Gateway cache (since
  2.8.0).
- `dataFrom.extract.property` can select a property containing nested JSON
  (since 2.8.0).

## Keeper

- Secrets can be retrieved by ID or name (since 2.4.0).
- `provider_api_calls_count` reports Keeper API-call volume (since 2.6.0).

## Devolutions Server

The provider is available (since 1.3.0) and can address entries by name (since
2.4.0).

## Doppler

- OIDC authentication is supported (since 1.2.0).
- Provider errors include the HTTP status for better diagnostics (since
  2.7.0).

## Conjur

- Unimplemented `PushSecret` and `DeleteSecret` operations return explicit
  errors (since 2.4.0).
- Certificate-based authentication is supported (since 2.8.0).

## Pulumi

Pulumi authentication supports OIDC (since 2.5.0).

## GitHub

- Provider errors are surfaced (since 0.17.0).
- `GithubProvider.orgSecretVisibility` configures organization-secret
  visibility (since 2.3.0).
- Updating an organization secret preserves its selected repositories (since
  2.7.0).

## Additional providers

- Nebius MysteryBox is available as a provider (since 2.1.0).
