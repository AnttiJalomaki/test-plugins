# Security, Resource Access, and Multitenancy

Use this reference for Security plugin upgrades, authentication mechanisms,
certificates, audit controls, centralized resource authorization, tenant
isolation, remote metadata, and plugin-specific multi-tenant constraints.

## Certificates, authentication, and password storage

### Certificates

- The Security plugin adds certificate hot reload and authority-certificate
  validation in 2.19.0, with an option to skip distinguished-name validation
  during hot reload.
- Security can authenticate from X.509 v3 Subject Alternative Name extensions
  in 3.4.0.
- `securityadmin.sh` adds `--timeout` and `-to` in 3.4.0.

### JWT, JWKS, and claims

- Roles can be stored in nested JWT claims from 3.1.0.
- JWT authentication can resolve the subject from a nested claim in 3.2.0.
- The JWT backend can consume a JWKS directly in 3.3.0.
- Nested JWT claims can be addressed with dot notation in 3.5.0.
- gRPC JWT header names are case-insensitive in 3.5.0.

### Client certificates, SPIFFE, and transport authentication

- Security 3.2.0 adds SPIFFE X.509 SVID authentication through
  `SPIFFEPrincipalExtractor`.
- Auxiliary transports can be configured for SSL only in 3.2.0.
- Client-certificate authentication adds
  `clientcert_auth_domain.http_authenticator.config.skip_users` in 3.3.0.
- gRPC gains JWT authentication through the Security plugin in 3.5.0.
- Security adds Basic authentication for gRPC in 3.6.0.

### Passwords and API keys

- Password-strength validation accepts `good` in 3.0.0.
- Security adds Argon2 password hashing in 3.2.0.
- Security 3.7.0 can issue long-lived API keys whose cluster and index
  permissions are attached directly to the key instead of inherited from user
  roles.
- API keys support expiration, synchronous cluster-wide revocation, automatic
  system-index protection, and create/list/revoke administration in
  Dashboards.

## Security configuration and permissions

### 3.x configuration changes

- The Security plugin removes its OpenSSL provider in 3.0.0.
- Whitelist settings are replaced by allowlist settings in 3.0.0.
- The `_cat/shards` action requires `cluster:monitor/shards` from 3.0.0.
- `ignore_hosts` accepts CIDR ranges in 3.0.0.
- Security resource settings are dynamically updateable in 3.4.0.
- Static and custom security configurations may overlap in 3.4.0, with static
  configuration taking precedence.
- `plugins.security.system_indices.indices` is deprecated in 3.4.0.

### Versioned security configuration

- An experimental versioned security-configuration system is available in
  3.2.0.
- Versioned security configuration adds View and Rollback APIs in 3.3.0.

### Cache and request checks

- Security adds a cache-flush endpoint for an individual user in 3.1.0.
- Changes to `plugins.security.cache.ttl_minutes` are picked up by a
  cluster-settings listener in 3.1.0.
- In 3.2.0, a query parameter can validate permission for a request without
  executing it.

### System and custom request controls

- Custom-attribute serialization is dynamically configurable in 3.3.0.
- Disabling `plugins.security.system_indices.enabled` permits plugin system
  requests in 3.3.0.
- The dynamic 3.5.0 setting `plugins.security.dls.write_blocked` blocks all
  writes when document-level restrictions apply.
- gRPC has circuit-breaker protection in 3.5.0.

## Audit logging

- Webhook audit-log sinks support Basic authentication in 3.4.0 through
  `plugins.security.audit.config.username` and
  `plugins.security.audit.config.password`.
- Audit logs gain configurable time zones in 3.5.0.
- Audit logs can include document contents for DELETE operations in 3.5.0.

## Centralized resource authorization

### Framework lifecycle

- The disabled-by-default 3.1.0 resource authorization framework centralizes
  sharing and access control in Security instead of reimplementing it in every
  plugin. Anomaly Detection is the first onboarded plugin.
- Security 3.2.0 provides an API to migrate resource-sharing data into the
  plugin.
- Resource sharing in 3.3.0 adds management APIs and a Dashboards interface,
  DLS-backed visibility filtering, persisted tenant and principal visibility,
  an explicit protected-resource list, and centralized access control for ML
  model groups.
- Flow Framework joins the centralized resource-sharing framework in 3.4.0,
  and one resource index can hold multiple sharable resource types.

### Sharing API changes

- A 3.4.0 sharing migration requires `default_owner` and a default access
  level.
- Update-sharing uses POST rather than PATCH in 3.4.0.
- The share and revoke Java APIs are removed in 3.4.0.
- Security resource configuration can set a default access level in 3.6.0.
- Resource providers in 3.6.0 can declare parent type and parent-ID fields for
  parent-child authorization.
- On-behalf-of token authentication no longer requires `encryption_key` in
  3.6.0.
- The resource configuration filename changes from
  `resource-action-groups.yml` to `resource-access-levels.yml` in 3.6.0.

## Remote metadata

### Storage and concurrency

- The 2.19.0 Remote Metadata SDK and repository wrapper let plugins keep
  metadata in external storage instead of system indexes on stateful nodes.
- The SDK adds global-resource support in 3.3.0.
- Put and delete operations add sequence-number and primary-term concurrency
  controls in 3.3.0.
- Put, update, delete, and bulk operations accept refresh policies and timeouts
  in 3.3.0.

### Encryption

- The Remote Metadata SDK can encrypt and decrypt customer data with
  customer-managed keys in 3.4.0 and assume a role for those key operations.

## Tenant isolation and Dashboards login

### Plugin resource isolation

- Tenant-ID isolation in 2.19.0 spans Flow Framework and ML Commons resources
  and operations, including connectors, models, tasks, deployment, prediction,
  agents, search, and configuration.
- DLS/FLS variables accept fallback values in 3.7.0.
- `opensearch_security.multitenancy.tenants.preferred` is dynamically
  updateable through the security configuration API in 3.7.0 without a
  Dashboards restart.
- `?auto_login=false` forces the login screen in 3.7.0.
- `opensearch_security.auth.default_redirect_auth_type` selects the default
  redirect authenticator in 3.7.0.

### Alerting and Anomaly Detection constraints

- With Alerting multi-tenancy enabled in 3.7.0, email, findings, chained
  actions, Job Scheduler indexes, and other unsupported actions are disabled.
- Pluggable-data-format domains reject non-PPL monitor CRUD in this mode.
- Anomaly Detection data sources for multi-tenant services disable default or
  flattened result indexes and historical analysis.
- Unsupported multi-tenant routes return HTTP 501.

## Notifications configuration

- Notifications 3.6.0 adds `multi_tenancy_enabled` and changes its settings
  prefix. Review existing notification configuration during upgrade.
