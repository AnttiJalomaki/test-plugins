# Security and Multitenancy

Use this reference for certificates, authentication, authorization,
resource-sharing, API keys, tenant isolation, audit controls, and secured gRPC.

## Certificates, identities, and authentication

### Security certificate hot reload

Since 2.19.0, the Security plugin can hot-reload certificates and validate
authority certificates. An option can skip distinguished-name validation
during the hot reload.

### Security cache and JWT operations

Since 3.1.0, Security can flush cache for one user and read roles from nested
JWT claims. A cluster-settings listener applies changes to
`plugins.security.cache.ttl_minutes`.

### Argon2 and workload identity

Since 3.2.0, Security supports Argon2 password hashing and SPIFFE X.509 SVID
authentication through `SPIFFEPrincipalExtractor`. Auxiliary transports can be
SSL-only, and JWT authentication can resolve its subject from a nested claim.

### Authentication and system-request controls

Since 3.3.0, the JWT backend can consume a JWKS directly. Client-certificate
authentication adds
`clientcert_auth_domain.http_authenticator.config.skip_users`. Custom-attribute
serialization is dynamically configurable, and disabling
`plugins.security.system_indices.enabled` permits plugin system requests.

### Security authentication and administration

Since 3.4.0, Security can authenticate X.509 v3 Subject Alternative Name
extensions, and `securityadmin.sh` accepts `--timeout` or `-to`. Webhook
audit-log sinks use Basic authentication through
`plugins.security.audit.config.username` and
`plugins.security.audit.config.password`.

### Tenant and login configuration

Since 3.7.0, DLS/FLS variables accept fallback values.
`opensearch_security.multitenancy.tenants.preferred` is dynamically updateable
through the Security configuration API without restarting Dashboards.
`?auto_login=false` forces the login screen, and
`opensearch_security.auth.default_redirect_auth_type` chooses the default
redirect authenticator.

## Configuration and permission validation

### Security configuration and request validation

OpenSearch 3.2.0 adds an experimental versioned Security configuration system.
Security can validate permission for a request through a query parameter
without executing it and provides an API to migrate resource-sharing data into
the plugin.

### Security configuration and resource access

Since 3.3.0, experimental versioned Security configuration provides View and
Rollback APIs. Resource sharing adds management APIs and a Dashboards UI,
DLS-backed visibility filtering, persisted tenant and principal visibility, an
explicit protected-resource list, and centralized access control for ML model
groups.

### Security configuration behavior

Since 3.4.0, resource settings are dynamically updateable. Static and custom
Security configurations may overlap, with static configuration taking
precedence. `plugins.security.system_indices.indices` is deprecated.

### Security write and audit controls

Since 3.5.0, dynamic `plugins.security.dls.write_blocked` blocks all writes when
document-level restrictions apply. Audit logs support configurable time zones
and can include document contents for DELETE. Nested JWT claims can be
addressed with dot notation.

## Centralized resource authorization

### Experimental centralized resource access

OpenSearch 3.1.0 introduces a disabled-by-default resource authorization
framework that centralizes plugin sharing and access control in Security.
Anomaly Detection is the first integrated plugin.

### Resource-sharing API changes

Since 3.4.0, Flow Framework participates in centralized sharing and access
control, and one resource index can contain multiple shareable resource types.
Migration requires `default_owner` and a default access level. Update-sharing
uses POST instead of PATCH, and the share and revoke Java APIs are removed.

### gRPC and resource authorization

Since 3.6.0, Security supports Basic authentication for gRPC. Resource
providers can declare parent type and parent-ID fields for parent-child
authorization. On-behalf-of token authentication no longer requires
`encryption_key`.

### Resource-access and notification configuration

Since 3.6.0, Security resource configuration can set a default access level.
The filename changes from `resource-action-groups.yml` to
`resource-access-levels.yml`. Notifications adds `multi_tenancy_enabled` and
changes its settings prefix; review existing settings during upgrade.

## gRPC protection and API keys

### Protected gRPC transport

Since 3.5.0, gRPC has circuit-breaker protection and JWT authentication through
Security. JWT header names are case-insensitive.

### Scoped API keys

Since 3.7.0, Security issues long-lived API keys whose cluster and index
permissions attach directly to the key instead of inheriting user roles. Keys
support expiration, synchronous cluster-wide revocation, automatic
system-index protection, and create/list/revoke administration in Dashboards.

## Plugin and tenant isolation

### Remote metadata and plugin multi-tenancy

Since 2.19.0, the Remote Metadata SDK and its repository wrapper let plugins
store metadata externally instead of in system indexes on stateful nodes.
Tenant-ID isolation covers Flow Framework and ML Commons resources and
operations, including connectors, models, tasks, deployment, prediction,
agents, search, and configuration.

### Multi-tenant plugin constraints

With Alerting multi-tenancy enabled in 3.7.0, email, findings, chained actions,
Job Scheduler indexes, and other unsupported actions are disabled.
Pluggable-data-format domains reject non-PPL monitor CRUD. Anomaly Detection
data sources for multi-tenant services disable default or flattened result
indexes and historical analysis. Unsupported routes return 501.
