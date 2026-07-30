# Authentication and access control

Use this reference for login behavior, OAuth, SAML, JWT, SCIM, service
accounts, identities, RBAC scopes and actions, audit policy, and Enterprise
access-control automation.

## Login and local authentication

- In 11.5.0, authentication gains a setting for the maximum failed login
  attempts before lockout.
- In 11.6.0, login-attempt validation can key on IP address.
- In 12.2.0, a setting can disable username-based brute-force protection.
- In 13.0.0, passwordless magic-link authentication is removed from both
  frontend and backend.

## API keys and service accounts

- In 11.6.0, startup migrates existing API keys to service accounts. Expect an
  authentication-object migration on the first startup.
- In 12.1.0, API-key endpoints and API-key authentication code are removed.
  Move clients to service accounts or another supported identity.

## OAuth and session behavior

- In 11.5.0, OAuth providers may authenticate token exchange with
  `client_secret_jwt`. Once refresh retries are exhausted, Grafana returns an
  error instead of silently continuing.
- In 11.6.0, `ssoSettingsSAML` becomes generally available and enabled by
  default.
- In 12.1.0, improved OAuth and SAML session handling becomes enabled by
  default. Authentication adds Azure/Entra workload identity and can extract
  user information from an access token.
- In 12.3.0, Enterprise query caching is disabled for data sources with
  `oauthPassThru=true`; this avoids sharing cached results across per-user
  OAuth identities.
- In 12.4.0, OAuth can verify ID-token signatures and can require a refresh
  token when `use_refresh_token` is enabled. SSO settings add PATCH.
- In 13.0.0, SSO logins remember the last organization used.

## SAML, LDAP, and enterprise SSO

- In 11.5.0, Enterprise SAML can set EntityID. Single logout carries NameID and
  SessionIndex for a complete SLO exchange.
- In 12.1.0, `ssoSettingsLDAP` is enabled by default.
- In 12.4.0, SCIM becomes generally available.
- In 13.0.0, Azure integrations support certificate authentication.

## JWT

- In 11.6.0, JWT authentication accepts `TlsSkipVerify`.
- In 12.0.0, JWT identities can map to Grafana organization roles.
- In 12.2.0, JWT accepts `tls_client_ca` and
  `jwk_set_bearer_token_file`.
- In 13.1.0, JWT accepts inline public keys.

## SCIM identity lifecycle

- In 12.1.0, Enterprise SCIM group PATCH can add or remove members and update
  `externalId`. User PATCH ignores unsupported fields. Team `externalId` is
  updateable.
- In 12.2.0, SCIM can reject login for users not provisioned by SCIM, and an
  update may set an empty `externalId`. DELETE now deletes a user rather than
  disabling the account.
- In 12.4.0, SCIM is generally available.

## Dashboard, snapshot, and library access

- In 11.5.0, create and delete snapshot operations receive separate RBAC roles
  and can be granted independently.
- In 12.0.0, `viewers_can_edit` is removed. Anonymous access enforces the
  configured Viewer organization role.
- In 12.0.0, dashboard endpoints under `/apis` perform fine-grained access
  checks.
- In 12.1.0, library-panel RBAC is generally available and enabled by default;
  the `libraryPanelRBAC` flag is removed.
- From 12.3.2 within 12.3.0, dashboard API requests enforce previously missing
  scope checks. Avatar endpoints require sign-in and honor their timeout, so
  anonymous avatar retrieval no longer works.
- In 13.0.0, pushing to Grafana Live is protected by RBAC.

## Data-source and plugin permissions

- In 11.6.0, plugin roles may include `plugins:write`. Drilldown access
  requires `datasources:explore`.
- In 11.6.0 Enterprise, a data-source query requires `query`; `read` no longer
  satisfies that authorization check.
- In 12.0.0, data-source label-based access control is available as a
  self-service public preview.
- In 12.1.0 Enterprise, data-source LBAC rules can filter by team.
- In 12.2.0, the plugin basic-role seeder stops granting plugin-app access.
- In 12.3.0, correlations no longer accept `org_id=0`; use a concrete
  organization ID for records and requests.

## Alerting permissions

- In 12.0.0, Alertmanager requests can provide `reqAction` for RBAC checks.
- In 12.4.0, Enterprise alert enrichment gets separate read and write
  permissions, and template testing gets a dedicated permission.
- During the 13.0-upgrade,
  `GET /api/alertmanager/grafana/api/v2/status` changes from
  `alert.notifications:read` to
  `alert.notifications.system-status:read`. Add the action to custom roles;
  administrators inherit it through `fixed:alerting.notifications:writer`.
- In 13.0.0, managed routes have access control. Provisioning can use
  resource-specific permissions and protected fields; notification APIs
  enforce provenance permissions.

## Custom-role migration

The 13.0-upgrade tightens role validation. Create, update, delete, or assignment
operations can fail when API-, Terraform-, or file-provisioned roles contain
deprecated permissions.

- A global role cannot carry a data-source UID scope such as
  `datasources:uid:<uid>`. Recreate it with a new UID as a non-global role;
  existing role scope cannot be changed in place. Set `datasource_type` on
  data-source permission resources where possible.
- Remove `fixed:annotations.dashboard:writer`,
  `fixed:annotations.dashboard:reader`, and `annotations:type:dashboard`.
  Dashboard annotations use dashboard or folder View/Edit/Admin permissions.
- Replace `annotations:*` with `annotations:type:organization` for
  organization annotations and dashboard/folder permissions for dashboard
  annotations.

## Role writes and maintenance

- In 12.3.0, RBAC writes persist action sets rather than expanded actions.
  Role-management clients should retain action-set references.
- In 12.4.0, `grafana cli admin flush-rbac-seed-assignment` clears seeded RBAC
  assignments.
- In 13.0.0 Enterprise, `/access-control/assignments/search` and the
  `IncludeMapped` parameter on
  `GET /access-control/users/{userId}/roles` are removed. Stop sending the
  deprecated role version on writes; Grafana increments it.
- In 13.0.0, Usage Insights changes dashboard and data-source identifiers from
  numeric IDs to UIDs.

## Audit controls

- In 12.2.0 Enterprise, audit settings can control recording of data-source
  query request and response bodies.
- In 12.4.0 Enterprise, Loki audit delivery gains retry and timeout settings.
- In 13.0.0 Enterprise, query request and response bodies are excluded from
  audit events by default. Opt in explicitly where required, accounting for
  the sensitivity and volume of those bodies.

## Cloud migration authorization

- In 11.5.0, the Cloud Migration assistant has a dedicated RBAC role.
- In 12.4.0, the feature toggle is replaced by a configuration option that can
  disable Cloud Migrations.

## Restricted-recipient and secret controls

- In 11.5.0, Enterprise reporting can restrict recipient email domains.
- In 13.1.0, alerting can limit email contact-point recipients to organization
  members. Enterprise reporting adds its own organization-member recipient
  limit.
- In 13.1.0, the AWS Secrets Keeper UI supports guided create, edit, activate,
  deactivate, and delete operations.
