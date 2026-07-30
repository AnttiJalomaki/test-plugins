# Identity, Access Control, and Enterprise Operations

Use this reference for login and token behavior, SSO and SCIM, RBAC migrations,
audit settings, Cloud Migrations authorization, and Enterprise controls.

## Login protection and session behavior

- A configurable maximum-login-attempt setting is available from 11.5.0.
- Login-attempt validation can include the client IP address, and JWT
  authentication accepts `TlsSkipVerify` (since 11.6.0).
- A setting can disable username-based brute-force login protection (since
  12.2.0). Review the threat model before disabling it.
- Improved OAuth and SAML session handling is enabled by default from 12.1.0.
- Passwordless magic-link authentication is removed from frontend and backend
  in 13.0.0. SSO logins remember the user's last organization.

## OAuth, JWT, and SAML

- OAuth providers can use `client_secret_jwt` for token exchange (since
  11.5.0). Exhausted refresh retries now return an error instead of silently
  continuing.
- Enterprise SAML accepts a configurable EntityID, and single logout includes
  NameID and SessionIndex for the SLO exchange (since 11.5.0).
- `ssoSettingsSAML` is generally available and enabled by default (since
  11.6.0).
- `ssoSettingsLDAP` is enabled by default (since 12.1.0).
- JWT identities can map to Grafana organization roles (since 12.0.0).
- Authentication adds Azure/Entra workload identity and can extract user
  information from access tokens (since 12.1.0).
- JWT supports `tls_client_ca` and `jwk_set_bearer_token_file` (since 12.2.0).
- OAuth can validate ID-token signatures and require refresh tokens when
  `use_refresh_token` is enabled (since 12.4.0). SSO settings expose a PATCH
  endpoint.
- Azure integrations support certificate authentication (since 13.0.0).
- JWT supports inline public keys (since 13.1.0).

## API keys and service accounts

Grafana migrates existing API keys to service accounts at the first 11.6.0
startup. Plan for the authentication-object migration and revalidate ownership
and automation. API-key endpoints and the API-key authentication
implementation are removed in 12.1.0; clients must use supported service
account tokens before that upgrade.

## RBAC and scoped authorization

- Cloud Migrations is enabled by default in 11.5.0, and the migration
  assistant has a dedicated role.
- Snapshot creation and deletion have separate RBAC roles (since 11.5.0).
- Plugin roles can contain `plugins:write`; Drilldown access requires
  `datasources:explore` (since 11.6.0).
- Enterprise data-source queries require `query`; `read` is no longer accepted
  as an alternative (since 11.6.0).
- Kubernetes dashboard routes under `/apis` perform fine-grained access checks
  (since 12.0.0).
- Self-service data-source label-based access control enters public preview in
  12.0.0. Enterprise LBAC rules can filter by team from 12.1.0.
- The plugin basic-role seeder no longer grants plugin-app access (since
  12.2.0).
- Dashboard API calls enforce scope checks from 12.3.2. Avatar requests require
  sign-in and honor their timeout; anonymous avatar fetches stop working.
- Correlations reject `org_id=0` from 12.3.0. Store and request a concrete
  organization ID.
- RBAC writes store only action sets from 12.3.0. Role automation should
  preserve action-set references instead of expecting expanded actions.
- `grafana cli admin flush-rbac-seed-assignment` flushes seeded RBAC
  assignments (since 12.4.0).
- Grafana Live push operations require RBAC (since 13.0.0).

## Grafana 13 custom-role migration

The `13.0-upgrade` applies stricter validation to role creation, updates,
deletion, and assignment. Terraform, provisioning, and API-managed roles can
fail if they still include deprecated permissions.

A global role cannot carry a data-source UID scope such as
`datasources:uid:<uid>`. Recreate it as a non-global role with a new UID
because an existing role's scope cannot be changed in place. Set
`datasource_type` on data-source permission resources where possible.

Remove `fixed:annotations.dashboard:writer`,
`fixed:annotations.dashboard:reader`, and `annotations:type:dashboard`. Use
dashboard or folder View/Edit/Admin permissions for dashboard annotations.
Replace `annotations:*` with `annotations:type:organization` for organization
annotations and dashboard/folder permissions for dashboard annotations.

Grafana 13.0.0 also removes `/access-control/assignments/search` and the
`IncludeMapped` parameter from `GET /access-control/users/{userId}/roles`.
Stop sending the deprecated role version on writes; Grafana increments it.
Usage Insights changes data-source and dashboard identifiers from numeric IDs
to UIDs.

## SCIM user, group, and team lifecycle

- Enterprise SCIM group PATCH supports adding or removing members and changing
  `externalId`; user PATCH ignores unsupported fields. Team `externalId` is
  mutable (since 12.1.0).
- Enterprise 12.2.0 can reject login for users not provisioned by SCIM.
  Updates may set an empty `externalId`, and SCIM DELETE now deletes a user
  rather than disabling it.
- SCIM is generally available from 12.4.0.

## Data access and cache isolation

Enterprise data-source caching is disabled when `oauthPassThru=true` (since
12.3.0), preventing per-user OAuth credentials from being combined with shared
cached query results.

## Audit configuration

- Enterprise 12.2.0 adds controls for recording data-source query request and
  response bodies.
- Enterprise audit delivery to Loki can configure retry count and timeout
  (since 12.4.0).
- Enterprise 13.0.0 disables data-source request and response bodies in audit
  logs by default. Explicitly opt back in only when the data exposure is
  acceptable and the bodies are operationally required.

## Cloud Migrations

- Cloud Migrations and its dedicated assistant role are available by default
  from 11.5.0.
- Mute timings are dependencies of notification policies during migration, so
  both resources move together (since 12.1.0).
- The Cloud Migrations feature toggle is removed in 12.4.0. Use the
  configuration setting when the feature must be disabled.

## Enterprise reporting and secrets

- Reporting adds an allowed-recipient-domain setting, includes its API server
  by default, and deprecates internal IDs (since 11.5.0).
- Report email subjects are configurable (since 11.6.0).
- Reporting supports schema V2 dashboards from 12.3.0.
- Retries are productized, stabilized PDF rendering stops using
  `newPDFRendering`, and schema-V2 forms can edit variables (since 12.4.0).
- PDF header toggles, configurable footers, and a readiness observer arrive in
  13.0.0.
- URL-based backend rendering and organization-member recipient limits arrive
  in 13.1.0.
- The AWS Secrets Keeper UI supports guided creation, editing, activation,
  deactivation, and deletion (since 13.1.0).

## Validation checklist

After an identity or authorization upgrade, exercise:

- password, OAuth, SAML, JWT, service-account, and anonymous flows;
- refresh failure, logout, last-organization, and brute-force behavior;
- custom and seeded roles, action sets, scopes, and protected fields;
- dashboard, avatar, snapshot, migration, plugin, Live, and data-source access;
- SCIM create, PATCH, login, external-ID, team, group, and DELETE semantics;
- audit-body defaults, delivery retries, and recipient restrictions.
