---
name: keycloak-knowledge-patch
description: Keycloak
version: 26.7.0
license: MIT
metadata:
  author: Nevaberry
---


# Keycloak Knowledge Patch

Use this skill when implementing, configuring, operating, extending, or upgrading
Keycloak. Determine the deployed server, adapter, Operator, and client-library
versions before applying version-dependent guidance. Prefer the project's
manifests, configuration, code, and tests when they establish different behavior.

## Reference index

| Reference | Topics |
| --- | --- |
| [Authentication and token exchange](references/authentication-and-token-exchange.md) | Browser flows, passkeys, LoA/ACR, token exchange, OID4VCI, ID-JAG, SSF, AuthZEN, delegation, and parameterized scopes |
| [Identity, brokering, and organizations](references/identity-brokering-and-organizations.md) | Redirects, CORS, logout, broker tokens, SCIM, organization roles, membership claims, and realm discovery |
| [Admin and account APIs](references/admin-and-account-apis.md) | Client Admin API v2, SCIM-style filtering, Account REST gates, application discovery, and consent CRUD |
| [Server configuration and operations](references/server-configuration-and-operations.md) | Environment variables, optimized builds, health and queues, truststores, datasources, clustering, Operator installation, and Kubernetes |
| [Deployment and storage migrations](references/migration-deployment-and-storage.md) | Hostname/proxy, trust, passwords, sessions, caches, metrics, transactions, database indexes, containers, LDAP, and removed runtime components |
| [Client, theme, and extension migrations](references/migration-clients-themes-and-extensions.md) | Themes, JavaScript, User Profile, token shapes, representations, logout, SPIs, events, registration, organizations, X.509, and feature removals |

## Working method

1. Identify the exact Keycloak server and Operator versions from the image,
   deployment manifest, or build metadata.
2. Identify separately versioned consumers such as `keycloak-js`, Java
   extensions, themes, adapters, and generated Admin clients.
3. Load the reference that matches the task; load both migration references
   before a major-version upgrade.
4. Treat preview and experimental feature flags as explicit opt-ins. Confirm
   that the profile enables the feature before relying on its endpoints or
   data model.
5. Test authentication, logout, refresh, proxy, health, and rolling-upgrade
   behavior at the protocol boundary rather than relying only on startup.
6. Preserve exact configuration spelling. Keycloak distinguishes build-time
   from runtime options and provides special environment-variable forms where
   normalization or expression expansion would change a value.

## Breaking changes and deprecations

### Preserve sessions during a 25-to-26 upgrade

- Upgrade through 25 with `persistent-user-sessions` enabled on that first
  upgrade when online sessions from 24 must survive.
- Confirm that the sessions already live in remote Infinispan or embedded-cache
  JDBC persistence. Enabling persistence later cannot safely merge the two
  populations.
- Expect 26 to clear caches when marshalling changes to Protostream. A direct
  upgrade that skips the persistence migration loses sessions.

### Replace removed hostname and proxy configuration

- Configure hostname v2 with `hostname`; use a full URL when scheme, port, or
  path matters.
- Give `hostname-admin` a full URL.
- Replace `proxy` with exactly one trusted `proxy-headers` format and set the
  required hostname and HTTP options.
- Enable dynamic backchannel resolution only with
  `hostname-backchannel-dynamic=true` and a full frontend URL.

### Move caches and transactions to runtime-safe settings

- Pass `cache`, `cache-stack`, and `cache-config-file` at runtime, not during
  image build.
- Account for `transaction-xa-enabled=false` as the default. With multiple
  datasources, permit at most one non-XA datasource and configure the others
  for XA.
- Bound each persisted session cache to an equivalent of 10,000 entries with
  one owner when replacing the standard cache XML.

### Migrate browser clients and themes

- Import browser code from `keycloak-js` or `keycloak-js/authz`; do not use
  deep package paths, a server-hosted script, or the removed UMD/global build.
- Pass adapter configuration explicitly, run in a secure context, and await
  `login()`, `createLoginUrl()`, and `createRegisterUrl()`.
- Move welcome themes to PatternFly 5, Account Console themes to
  `parent=keycloak.v3`, and shared third-party paths to `vendor/...`.
- Test custom templates against FreeMarker 2.3.32 behavior.

### Update authentication and logout integrations

- Use OIDC RP-Initiated Logout; remove the legacy logout `redirect_uri`
  behavior and its deleted SPI switches.
- Use `prompt=create` for registration and the supported
  `/forgot-credentials` authorization-path variant for credential reset.
- Never deep-link to `/login-actions` or `/broker`.
- Replace the boolean WebAuthn discoverable-credential setting with
  `required`, `preferred`, or `discouraged`.

### Update extensions and representations

- Treat `UserRepresentation.getAttributes()` as custom attributes only; use
  dedicated root properties or server-side `getRawAttributes()`.
- Query identity providers from their dedicated endpoint because ordinary
  realm representations no longer embed them.
- Do not restart a request's `KeycloakSession` transaction. Give asynchronous
  work its own session and transaction lifecycle.
- Replace removed token convenience methods with `exp`, `nbf`, and `iat`, and
  pass `Config.Scope` to `EnvironmentDependentProviderFactory.isSupported`.

### Remove obsolete switches and bundled dependencies

- Remove `token-exchange-external-internal:v2` and the persistent-session batch
  options.
- Replace `dynamic-scopes` with `parameterized-scopes`.
- Install an Oracle JDBC driver explicitly when required; do not rely on
  removed GELF, BOM, test-helper, JEE admin-client, or legacy LinkedIn OAuth
  artifacts.
- Replace deprecated bootstrap administrator variables with
  `KC_BOOTSTRAP_ADMIN_USERNAME` and `KC_BOOTSTRAP_ADMIN_PASSWORD`.

## Security-critical quick reference

### Redirects, origins, and logout

- Match redirect URIs exactly and case-sensitively unless the registered URI
  has a trailing wildcard.
- Force exact matching when a wildcard request contains userinfo or `/../`.
- Never use a full `*` redirect pattern in production.
- Treat token-embedded Web Origins as adapter behavior, not portable OIDC
  metadata.
- Validate `post_logout_redirect_uri`; with logout confirmation enabled, render
  it as a continuation rather than automatically redirecting.

### Token exchange

- Use standard token exchange v2 for confidential authenticated requesters and
  explicitly enable *Standard token exchange* on each requester.
- Exchange only same-realm Keycloak access tokens; do not send `resource`.
- Treat repeated `audience` values as filters, never as additions.
- Bind sender-constrained exchanges to the original client and require the
  matching DPoP proof or mTLS certificate.
- Restrict legacy exchange permissions to explicitly trusted clients because
  its external JWT path does not validate `aud`.

### Authentication assurance

- Order conditional LoA subflows from lowest to highest.
- Use an essential `claims` request when failure is required; `acr_values`
  remains non-essential.
- Protect browser-carried assurance requests with PAR or a request object and
  verify the returned `acr`.
- Place the session limiter after the user is known and avoid rechecking normal
  SSO-cookie reuse.

### Secrets, trust, and X.509

- Use `KCRAW_` for literal secrets containing dollar signs; do not define the
  matching `KC_` form at the same time.
- Never put secrets in build options because every build option is persisted
  in plaintext.
- Trust the authenticator CA for direct WebAuthn attestation.
- Add the exact *Certificate Authority subject DN* to X.509 client credentials;
  migrate away from regex subject matching and `ssl-cert-chain-prefix`.
- Allow at least 86 characters in downstream stores for newly generated client
  secrets.

## High-value features

### Declarative client administration

- Enable `client-admin-api:v2` for strictly validated OIDC and SAML client
  management through REST, Java, generated JavaScript, CLI, or Operator custom
  resources.
- Use the management-interface OpenAPI description and treat its representations
  as declarative contracts.
- Apply `q` filters before `fields` projection; reject unknown fields,
  malformed syntax, and unsupported ordering comparisons.

### Organizations and provisioning

- Delegate coarse-grained organization access with `manage-organizations`,
  `view-organizations`, and `query-organizations`.
- Add user-view permission before listing members, and use fine-grained
  organization resources for per-organization access.
- Enable the organization membership mapper's group-role option when realm and
  client roles must appear per organization in tokens.
- Enable `scim-api` to use preview user/group provisioning, PATCH, filtering,
  pagination, extensions, and schema discovery.

### Policy and event protocols

- Enable `authzen` to evaluate single or batched subject/resource/action
  requests against authorization policies.
- Enable `ssf` to transmit signed CAEP or RISC Security Event Tokens through
  push or poll streams backed by durable, cluster-aware delivery.
- Enable `identity-assertion-jwt` only when Keycloak is the receiving
  authorization server; the other ID-JAG roles are not implemented.
- Enable `token-exchange-delegation` for consented, refresh-time-reassessed
  delegation.

### Operational resilience

- Route traffic with `/health/ready`; startup and liveness may be UP while
  asynchronous initialization is still incomplete.
- Set `http-max-queued-requests` to reject excess queued work with HTTP 503.
- Give provider JARs deterministic modification times before optimized builds.
- Use the `stateless` feature for preview multi-cluster v2 and plan around its
  database-backed invalidation outbox rather than external Infinispan or
  fencing.
