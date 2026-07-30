# Authentication, Identity, and Policy

## Identity entities, aliases, and SCIM

- The disabled-by-default `force_identity_deduplication` activation flag
  resolves duplicate entities and groups by renaming them.
- Identity entity merges require `sudo`.
- Rendered identity templates reject wildcards.
- Enterprise AppRole, AWS, certificate, GitHub, LDAP, Okta, RADIUS, SCEP, and
  userpass configuration accepts `alias_metadata`; it populates alias custom
  metadata.
- Enterprise SCIM 2.0 server support is beta. It provisions entities, aliases,
  and groups from an external manager.
- SCIM user PATCH can include multiple changes, update metadata, names, or
  active status, and use explicit paths. Group member removal can address
  `members[value eq "id"]`.

## LDAP, RADIUS, and certificate auth

- LDAP login rejects an empty password and fails if user-DN search finds
  multiple entries. An option permits `sAMAccountName` login when `upndomain`
  is configured.
- `deny_null_bind` is deprecated because empty-password logins are always
  rejected.
- LDAP auth can configure a different URL for root-credential rotation.
  MFA/TOTP is enforced when `username_as_alias` is enabled.
- RADIUS `case_insensitive_names` prevents case-only username collisions and
  enables case-insensitive matching.
- `x_forwarded_for_client_cert_header` accepts an RFC 9440 colon-wrapped Base64
  client certificate.
- `enable_metadata_on_failures` can include certificate metadata in failed
  login responses and audit records.
- Certificate roles accept `allowed_organizations`. Non-CA login matching
  compares certificate equality, renewal requires the certificate attached to
  the session, and role-based quotas apply to certificate auth.

## Azure, Kubernetes, and cloud auth

- Azure login requires configured `resource_group_name`, `vm_name`, and
  `vmss_name` to equal the token claims.
- Azure auth requires a bound group or service-principal ID.
- From 2.0, stored `auth/azure/config` values take precedence over `AZURE_*`
  environment variables.
- The Azure OIDC provider can retrieve groups from the Azure Graph API.
- Kubernetes auth warns about role audiences when roles are created or
  updated; review audience settings rather than ignoring the warning.
- Enterprise GUI workload identity federation configuration covers AWS,
  Azure, and GCP integrations.

## Auto-auth and Cloud Foundry

- `enable_reauth_on_new_credentials` lets supported auto-auth methods
  authenticate again when credentials change.
- Certificate auto-auth watches its certificate and private-key files when
  that option is enabled.
- Cloud Foundry auth initializes its CF client only on a configuration write or
  login that needs it. `force_new_client` creates a client for every login
  instead of reusing the cached client.

## MFA and login UX

- Enterprise users can self-enroll in login MFA TOTP using a QR code and secret
  generated during login. Set `enable_self_enrollment` on the TOTP login-MFA
  method.
- Community Edition GUI can list and add TOTP accounts, reveal codes hidden by
  default, and show expiry timers.
- Enterprise can configure default and backup login-form auth methods.
  `/vault/auth?with=` now means only an auth mount path, renders a simplified
  form, and is not rewritten when another method is chosen.

## SPIFFE and agent authorization

- Enterprise SPIFFE auth authenticates JWT- or X.509-based SPIFFE IDs.
- Authenticated workloads can request JWT-SVIDs from Vault.
- Enterprise agent authorization adds an Agent Registry and lets Vault act as
  an OAuth resource server. Once configured, OAuth 2.0 JWTs can authorize
  requests for registered agent entities without a Vault token.
- Native agent support is a public beta for all customers by 2.0.3 rather than
  only an Enterprise beta.

## SAML protection

Set `VAULT_SAML_DENY_INTERNAL_URLS` to prevent `idp_metadata_url`,
`idp_sso_url`, and `acs_urls` from resolving to internal IP addresses.

## Policies, quotas, and authorization

- For `allowed_parameters` and `denied_parameters`, per-element matching uses
  “contains all” behavior. `VAULT_NEW_PER_ELEMENT_MATCHING_ON_LIST` opted into
  it before exact whole-list comparison was retired in 1.21.
- `resultant-acl` merges segment-wildcard (`+`) rules with prefix rules in
  `glob_paths`.
- Enterprise soft-mandatory Sentinel overrides honor the policy override flag.
- Enterprise rate-limit quotas accept `group_by` for entity-based and
  collective grouping.
- Enterprise SCEP roles enforce `token_bound_cidrs`.

## Namespace and policy UI

- The Enterprise namespace picker searches, filters, and navigates without
  reauthentication.
- The GUI can create a namespace from a guided questionnaire, then continue
  setup through GUI, CLI, or Terraform.
- The GUI visual policy editor generates ACL snippets.
- An Enterprise 2.0 Endpoint Governing Policy can block root-token GUI access
  to a child namespace through `sys/internal/ui/mounts`; use CLI/API or permit
  that endpoint.
