# Authentication, identity, and policy

Use this reference for auth-method behavior, identities and aliases, ACL evaluation, privileged endpoints, MFA, workload identity, OAuth authorization, SCIM provisioning, and namespace access.

## Privileged system operations

- `sys/generate-root`, `sys/replication/dr/secondary/generate-operation-token`, and `sys/rekey` authenticate callers by default. A root token generated on the primary can authenticate to a DR secondary.
- Legacy unauthenticated behavior can be explicitly enabled with `enable_unauthenticated_access = ["generate-root", "generate-operation-token", "rekey"]`. Treat this as a security exception, not a compatibility default.
- Identity entity merges require `sudo`.
- A root token can relock an Enterprise namespace.

## Identity de-duplication and templates

- `force_identity_deduplication` is disabled by default. When activated, Vault resolves duplicate entities and groups by renaming them; assess downstream references before enabling it.
- Wildcards are rejected in rendered identity templates.
- Enterprise AppRole, AWS, certificate, GitHub, LDAP, Okta, RADIUS, SCEP, and userpass configuration accepts `alias_metadata`, which populates alias custom metadata (since 1.21).

## Policy parameter matching

- `VAULT_NEW_PER_ELEMENT_MATCHING_ON_LIST` opted into “contains all” matching for list-valued `allowed_parameters` and `denied_parameters` during the transition.
- Exact-match list comparison is retired in 1.21.x. Policies must use per-element matching; test deny precedence and multi-value requests during upgrades.
- `resultant-acl` merges segment-wildcard (`+`) paths into `glob_paths` alongside prefix rules, producing a more complete view of glob-style permissions.
- Enterprise soft-mandatory Sentinel requests can proceed when the policy override flag is set; an earlier behavior ignored the override.
- Mount tuning can unset `allowed_response_headers` (since 1.21).
- Enterprise rate-limit quotas accept `group_by` for entity-based and collective grouping modes (since 1.20). Confirm the grouping identity and expected shared budget before rollout.

## Forwarded authentication data

- Before forwarding a request to a plugin backend, Vault strips Vault tokens from `Authorization` unless that header is explicitly configured as a passthrough request header.
- Certificate auth behind a proxy accepts RFC 9440 colon-wrapped Base64 certificates through `x_forwarded_for_client_cert_header`.
- `enable_metadata_on_failures` can put client-certificate metadata in failed-login responses and audit records. Review disclosure and audit-HMAC requirements before enabling it.

## LDAP and directory authentication

- LDAP login rejects empty passwords and errors when user-DN search produces multiple entries.
- When `upndomain` is configured, an option permits login with `sAMAccountName`.
- `deny_null_bind` is deprecated and ineffective because empty-password login is always denied; remove it (`upgrade-safety`).
- LDAP auth can configure a separate URL for root-credential rotation when that service is reached at a different endpoint.
- MFA/TOTP is enforced when `username_as_alias` is enabled (since 1.21).
- Active Directory root-password rotation accepts `schema`, defaulting to `openldap` for compatibility.
- RADIUS `case_insensitive_names` avoids case-only collisions and enables case-insensitive username handling.

## Azure authentication

- Login requires configured `resource_group_name`, `vm_name`, and `vmss_name` values to match token claims.
- Auth configuration requires at least a bound group or service-principal ID from 1.20; update unbound roles before upgrading.
- From 2.0, stored values at `auth/azure/config` take precedence over `AZURE_*` environment variables. Move intended overrides into stored configuration (`upgrade-safety`).
- The Azure OIDC provider can fetch groups through the Azure Graph API.

## Certificate auth

- Certificate-auth roles accept `allowed_organizations` (since 1.21).
- Non-CA login matching compares certificate equality.
- Renewal requires the certificate attached to the session.
- Role-based quotas apply to certificate auth.

## Auto-auth and Cloud Foundry

- `enable_reauth_on_new_credentials` lets supported auto-auth methods authenticate again when credential material changes. Certificate auto-auth watches its certificate and key files when enabled.
- The Cloud Foundry auth plugin initializes its client only for configuration writes or logins that need it. `force_new_client` creates a fresh client per login instead of sharing the cached client (since 2.0).

## Kubernetes auth

Kubernetes auth roles emit audience warnings from 1.20. Review configured audiences when creating or updating roles, and verify the presented service-account token against the intended audience.

## Login MFA self-enrollment

Enterprise users can self-enroll in login MFA TOTP. Set `enable_self_enrollment` on the TOTP login-MFA method; the login flow returns the QR code and secret used for enrollment.

## SPIFFE workloads

- Enterprise SPIFFE auth accepts JWT- or X.509-based SPIFFE IDs (since 1.21).
- From 2.0, authenticated workloads can request JWT-SVIDs, so Vault can issue SPIFFE workload identities as well as authenticate existing ones.

## Agent Registry and OAuth resource server

Vault 2.0 adds an Agent Registry and OAuth resource-server capability. Registered agent entities can present OAuth 2.0 JWTs directly to authorize Vault requests without first exchanging them for a Vault token. The feature started as an Enterprise beta and is available to all customers as a public beta by 2.0.3; verify beta constraints in the installed patch.

## SCIM provisioning

Enterprise beta SCIM 2.0 support provisions entities, aliases, and groups from an external identity system.

- User `PATCH` requests may contain multiple operations.
- Operations can change metadata, names, or active status and can specify explicit paths.
- Group-member removal can address a member with a path such as `members[value eq "id"]`.

## SAML URL protection

Set `VAULT_SAML_DENY_INTERNAL_URLS` to prevent `idp_metadata_url`, `idp_sso_url`, and `acs_urls` from resolving to internal IP addresses. Validate the IdP and ACS topology before enabling it so legitimate internal endpoints are not unexpectedly blocked.

## Centrify support

The Centrify auth plugin is no longer officially supported. Deployments that still use it need a supported replacement before upgrading (`upgrade-safety`).

## Namespace and EGP behavior

- Enterprise GUI namespace navigation can search, filter, and enter namespaces without requiring reauthentication.
- An open 2.0 issue can deny root-token GUI access to a child namespace protected by an Endpoint Governing Policy when the GUI requests `sys/internal/ui/mounts`.
- For that issue, use the CLI or API, or explicitly allow `sys/internal/ui/mounts` in the EGP (`upgrade-safety`).

## Review checklist

1. Resolve the mount path, local flag, namespace, and plugin type.
2. Check claim bindings, aliases, certificate equality, MFA, audience, and forwarded-header behavior.
3. Re-evaluate list-valued policy rules using per-element semantics.
4. Confirm `sudo`, Sentinel override, EGP, and role-quota requirements.
5. Treat OAuth, SCIM, and agent authorization as beta where the installed release says so.
6. Exercise login, renewal, reauthentication after credential changes, and failed-login audit output.
