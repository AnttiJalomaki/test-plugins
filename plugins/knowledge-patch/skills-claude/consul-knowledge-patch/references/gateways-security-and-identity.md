# Gateways, Security, and Identity

## Validate KV Key Names

The key/value endpoint validates key names (since 1.22.0). This is a breaking
security change for keys that older deployments accepted even though their
names are invalid. Audit existing data paths and clients before upgrading.

`DisableKVKeyValidation` controls whether validation is disabled. Treat this
as a compatibility escape hatch and make the security tradeoff explicit.

## Use PKCE and JWT Client Authentication

PKCE is enabled by default for Consul UI OIDC login (since 1.22.0). OIDC
providers can also authenticate the client with a JWT assertion instead of a
client secret. Verify provider support and client-authentication configuration
when replacing a stored secret.

## Rely on Normalized Gateway Paths

API Gateway and terminating-gateway HTTP listeners normalize request paths
(since 2.0.0). Normalization prevents non-normalized paths from bypassing L7
intention RBAC checks. Test routing and intentions with the normalized form,
especially when existing clients emit unusual paths.

## Supply Gateway Certificates with SDS

API Gateway listeners can use a default SDS TLS certificate (since 2.0.0).
HTTP or TCP route services can override that certificate. When a route does
not supply an override, it inherits the listener's SDS cluster. Conflicting
override mappings are rejected.

Terminating-gateway upstream TLS also uses SDS, so updated certificates become
available without restarting the gateway.

## Limit API Gateway Upstreams

Set gateway-wide upstream defaults or route-service overrides for these limits
(since 2.0.0):

- `MaxConnections`
- `MaxPendingRequests`
- `MaxConcurrentRequests`

Use defaults for consistent protection and overrides only where a route
service has a justified capacity profile.

## Delegate Mesh CA Signing to CyberArk WIM

Enterprise can delegate service-mesh certificate signing to CyberArk Workload
Identity Manager, also known as Venafi Firefly (since 2.0.0). Select the
provider by setting `connect.ca_provider` to `"pan-distributed-issuer"`.

Treat the external issuer as part of the mesh certificate trust and
availability path.
