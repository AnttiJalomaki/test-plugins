# Integration Providers

Use this reference for provider-specific authentication, lookup, extraction,
diagnostic, and maturity-sensitive behavior outside the major cloud and Vault
families. Provider push semantics remain centralized in `push-secrets.md`.

## 1Password

### SDK provider

- An SDK-based 1Password provider is available (0.17.0).
- `GetSecretMap` has parity with the Connect provider (0.18.0).
- A vault can be selected by UUID (0.18.0).
- Native item IDs are supported (2.2.0).
- `GetAllSecrets` enables bulk selections that require the all-secrets operation
  (2.7.0).
- Each client receives a fresh provider instance, avoiding a race that could
  route operations to the wrong vault (2.5.0).

See `push-secrets.md` for SDK multi-field push behavior.

### Connect and authorization behavior

Authorization errors are retried (0.20.0), so transient authorization failures
can recover without a manual resource change. See `push-secrets.md` for Connect
write capability.

## Akeyless

- `azure_ad` Workload Identity authentication uses `serviceAccountRef` (2.8.0).
- `SecretStore` configuration accepts `ignoreCache` to bypass the Akeyless
  Gateway cache (2.8.0).
- `dataFrom.extract.property` can select a property whose value is nested JSON
  (2.8.0).

## Infisical

- Missing-secret and incorrect-authentication errors that were previously
  silent are surfaced (0.13.0).
- `data` references can address secrets within paths (0.17.0).
- Authentication methods are configurable (0.19.0).
- Kubernetes authentication can use a Client JWT as its Reviewer JWT token
  (0.20.0).
- `dataFrom.find.path` filters by secret path rather than secret name (2.2.0).
- Secret scopes accept an organization slug (2.7.0).
- Push and HTTP 404 absence semantics are documented in `push-secrets.md`.

## Delinea Secret Server

- Fetching handles secret values that are not JSON (0.18.0).
- Connections accept a domain field (0.20.0).
- A secret can be fetched by path (0.20.0).
- Provider connections configure TLS correctly (1.1.0).
- Access-token authentication is supported (2.8.0).

## BeyondTrust

- Provider configuration accepts an API-version parameter (0.14.0).
- Get-secret calls accept `decrypt` to request decryption (1.3.0).
- Secret creation supports API v3.2 (2.5.0).
- BeyondTrust WorkloadCredentials is available as a provider (2.7.0).

## Passbolt

- The provider honors its refresh interval (0.14.0).
- Passbolt V5 API support is available (2.2.0).
- A custom CA bundle or CA provider can establish trust (2.4.0).
- `ExternalSecret` accepts the `v5-custom-fields` resource type (2.6.0).

## GitHub

- GitHub provider errors are surfaced rather than hidden (0.17.0).
- Organization-secret visibility and selected-repository preservation during
  push are documented in `push-secrets.md`.

## Grafana

The service-account generator is documented in
`templates-generators-and-cli.md`; chart dashboard discovery is documented in
`helm-and-operations.md`.

## Barbican

Barbican is available as a provider (1.2.0). It supports `property` and
`extract`, and `find.name.regexp` is interpreted as a regular expression rather
than an exact name (2.8.0).

## Cloud.ru Secret Manager

Cloud.ru Secret Manager is supported (0.15.0), and its provider accepts paths
(2.2.0).

## Devolutions Server

Devolutions Server is available as a provider (1.3.0). Entries can be addressed
by name (2.4.0).

## Keeper

- Secrets can be looked up by ID or name (2.4.0).
- `provider_api_calls_count` exposes provider API-call volume (2.6.0).

## Doppler

- OIDC authentication is supported (1.2.0).
- Provider errors include the HTTP status, improving failure diagnosis (2.7.0).

## Pulumi

Pulumi authentication supports OIDC (2.5.0).

## Conjur

Conjur supports certificate-based authentication (2.8.0). See
`push-secrets.md` for its unsupported write operations.

## Nebius MysteryBox

Nebius MysteryBox is available as a provider (2.1.0).

## OVHcloud

OVHcloud is available as a provider (2.3.0).
