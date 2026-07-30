---
name: keycloak-knowledge-patch
description: Keycloak
version: 26.7.0
license: MIT
metadata:
  author: Nevaberry
---


# Keycloak Knowledge Patch

Use this skill for Keycloak design, implementation, migration, extension, and
operations work. Check the deployed server version and its enabled feature
profile before applying version-sensitive advice. Treat project manifests,
configuration, code, and observed behavior as authoritative when they differ.

## Reference index

| Reference | Topics |
|---|---|
| `references/identity-and-authentication.md` | Redirects, consent, logout, browser flows, LoA/ACR, passkeys, SAML, X.509, DPoP |
| `references/protocols-token-exchange-and-brokering.md` | Standard and legacy token exchange, brokering, OID4VCI, ID-JAG, SCIM, SSF, AuthZEN |
| `references/admin-account-and-organizations.md` | Client Admin API v2, Account REST, organization permissions and responses |
| `references/server-operator-and-observability.md` | Quarkus configuration, optimized builds, health, caches, datasources, Operator, metrics |
| `references/upgrades-extensions-and-themes.md` | Major-version migrations, Java/JS extensions, themes, features, removed integrations |

## Start with breaking changes

### Preserve sessions across the 25-to-26 transition

- To retain sessions originating on 24, first upgrade to 25 with preview
  `persistent-user-sessions` enabled.
- Only sessions already backed by remote Infinispan or embedded-cache JDBC
  persistence are migratable. Enabling persistence after the first upgrade
  cannot safely merge persisted and non-persisted sessions.
- Version 26 changes cache marshalling from JBoss Marshalling to incompatible
  Protostream and clears caches. A direct 24-to-26 upgrade loses sessions.
- In 26, sessions are persisted by default. Bound custom session caches like
  the standard configuration: 10,000 entries and one owner.

### Replace removed hostname, proxy, and truststore settings

- Hostname v2 is the default from 25 and the only hostname implementation in
  26. `hostname` accepts a host or full URL; `hostname-admin` requires a full
  URL. Separate path and port options are gone.
- Replace `proxy` with exactly one trusted `proxy-headers` format. Enable HTTP
  only where edge termination requires it.
- Dynamic backchannel URLs require `hostname-backchannel-dynamic=true` and a
  full frontend URL.
- Replace old truststore SPI and HTTPS truststore settings with
  `conf/truststores` or `truststore-paths`; use `tls-hostname-verifier`.

```bash
bin/kc.sh start \
  --hostname=https://sso.example.com:8543/auth \
  --proxy-headers=xforwarded \
  --http-enabled=true
```

### Move runtime cache options out of image builds

`cache`, `cache-stack`, and `cache-config-file` are runtime-only from 25.
Remove them from `kc.sh build`; otherwise the server can silently use runtime
defaults. Build options are persisted in plaintext and must never contain
secrets.

### Update browser JavaScript integration

- Import only from `keycloak-js` and `keycloak-js/authz`; deep package imports
  crossed the package-exports boundary in 24.
- The 26 server no longer serves the library and the UMD/global build is gone.
  Install it in the application and pass configuration explicitly.
- Run it in a secure context and await `login()`, `createLoginUrl()`, and
  `createRegisterUrl()`.

```javascript
import Keycloak from "keycloak-js";
const keycloak = new Keycloak({
  url: "https://sso.example.com",
  realm: "example",
  clientId: "web"
});
await keycloak.login();
```

## Feature and deprecation checks

### Pin versioned feature implementations

An enabled feature cannot also be disabled. An unversioned enabled name selects
the latest supported implementation, so use `name:vN` when upgrades must not
change it. Disable a versioned feature by its unversioned base name.

Important feature names and states:

| Capability | Configuration and boundary |
|---|---|
| Standard token exchange | `token-exchange-standard:v2`; enabled by default, but opt in each confidential requester |
| Token delegation | `token-exchange-delegation`; experimental parameterized `delegation` scope |
| Parameterized scopes | `parameterized-scopes`; replaces `dynamic-scopes` |
| Identity assertion JWT grant | `identity-assertion-jwt`; receiver role only |
| SCIM | `scim-api`; preview and outside the default profile |
| Shared Signals | `ssf`; experimental |
| AuthZEN | `authzen`; experimental |
| Client Admin API v2 | `client-admin-api:v2`; experimental |
| Multi-cluster v2 | `stateless`; preview |
| OID4VCI pre-authorized code | `oid4vc-vci-preauth-code` |
| Attestation-based client auth | `client-auth-abca` |

Remove obsolete `token-exchange-external-internal:v2` and persistent-session
batching switches. The OIDC *Bearer only* switch and OAuth 1.0a Twitter broker
are deprecated. Model service clients with no enabled grants and migrate
Twitter to a generic OAuth v2 provider.

## Authentication quick reference

### Redirects, CORS, and logout

- Redirect URIs match exactly and case-sensitively unless they have a trailing
  wildcard. Userinfo or `/../` in a requested URI forces exact matching.
  Never use the full `*` HTTP/HTTPS pattern in production.
- *Web Origins* are embedded in access tokens for Keycloak adapters to enforce;
  they are not portable OIDC metadata.
- A backchannel logout URL is used only when front-channel logout is disabled.
  Without one, the nonstandard Admin URL fallback works only with legacy Java
  OIDC adapters and the Elytron WildFly OIDC adapter.
- With *Logout confirmation* enabled, a validated
  `post_logout_redirect_uri` becomes a continuation control, not an automatic
  redirect. Version 26 requires standards-based RP-Initiated Logout.

### Conditional 2FA and assurance levels

- The default conditional 2FA branch combines *Condition - User Configured*
  and *Condition - credential*, allowing passwordless WebAuthn to skip a
  redundant second factor.
- Set disabled WebAuthn and recovery-code executions to *Alternative* to expose
  them through *Try Another Way* for configured users.
- Order conditional LoA subflows from lowest to highest. The first runs on
  initial authentication; `Max Age=0` makes that level valid only then.
- Essential `claims` requests must be satisfied; `acr_values` is advisory.
  Protect browser-carried requests with PAR or request objects and verify the
  returned `acr`.
- Place *User Session Count Limiter* after user identification and use one
  consistent configuration across applicable flows. It does not support CIBA.

## Token exchange quick reference

Standard V2 supports authenticated confidential requesters only. It exchanges a
same-realm Keycloak access token for an access token, ID token, or conditionally
a refresh token; it does not support RFC 8693 `resource`.

```bash
curl -u requester-client:secret \
  https://keycloak.example/realms/test/protocol/openid-connect/token \
  -d grant_type=urn:ietf:params:oauth:grant-type:token-exchange \
  -d subject_token="$SUBJECT_TOKEN" \
  -d subject_token_type=urn:ietf:params:oauth:token-type:access_token \
  -d requested_token_type=urn:ietf:params:oauth:token-type:access_token
```

- `scope` adds requester optional scopes; repeated `audience` values only
  filter resolved audiences and never create them.
- Require the requester in the subject token's `aud`, unless the token was
  issued to that client. Use `downscope-assertion-grant-enforcer` to keep
  requested scopes within the subject token's grants.
- DPoP- or mTLS-bound subject tokens may be exchanged only by their original
  client with the matching proof or certificate.
- An exchange never creates a user session. Original access-token revocation
  leaves exchanged access tokens active but revokes derived refresh-token
  chains and requester client sessions.
- Legacy V1 is disabled, deprecated, and isolated on FGAP V1. Its external JWT
  path does not validate `aud`; grant exchange permission only to explicitly
  trusted clients.

## API quick reference

### Client Admin API v2 filters

`GET /admin/api/{realmName}/clients/v2` uses `q` with `eq`, `ne`, `co`, `sw`,
`ew`, `pr`, Boolean operators, and parentheses. Strings are case-sensitive and
double-quoted; booleans are bare. Invalid syntax, unknown fields, and ordering
operators return 400. Filtering evaluates the full representation before
`fields` projection.

### Account API boundaries

- `ACCOUNT_API` gates sessions, credentials, UMA, organizations, verifiable
  credentials, applications, and consent operations with 404 when disabled.
- Profile root, locales, linked accounts, and groups are outside that check.
- Profile reads require `manage-account` or `view-profile`; writes require
  `manage-account` and validate in the `ACCOUNT` user-profile context.
- Application discovery unions online/offline sessions, consents, and
  always-visible clients; bearer-only clients are excluded.
- Consent POST and PUT are both upserts. Parameterized client scopes are
  rejected, and scope IDs must resolve to realm client scopes or the
  consent-required client itself.

## Runtime quick reference

### Preserve literal configuration

`KC_` values evaluate expressions and collapse `$$`. Use the matching `KCRAW_`
name for literal dollar characters; defining both forms for one key is an
error. Use `KCKEY_<suffix>` beside `KC_<suffix>` when normalized environment
names cannot represent the exact option key.

```bash
export KCRAW_DB_PASSWORD='my$$pa${vault}word'
export KC_MYKEY=debug
export KCKEY_MYKEY=log-level-package.class_name
```

### Build and startup safely

- With `start --optimized`, a repeated build option is accepted only when it
  matches the built value. Rebuild to change it.
- Normalize provider JAR modification times before `kc.sh build` in container
  images so runtime does not report false provider changes.
- Set `http-max-queued-requests`; excess requests receive 503. The default
  queue is unlimited.
- Async bootstrap can open listeners while readiness is DOWN. Route only on
  `/health/ready`, or use `--server-async-bootstrap=false`.
- Set container memory limits; the image defaults maximum heap to 70% of
  available container memory.

### Datasources, health, and observability

- Exclude optional additional datasources from health checks when their
  failure must not mark the deployment unhealthy.
- From 26, at most one datasource may be non-XA. Enable XA on the default and
  configure `quarkus.datasource.<name>.jdbc.transactions=xa` on additions.
- Health and metrics use management port `9000`; the legacy observability
  switch only temporarily restores application-listener placement.
- The 26.7 shutdown default is ten seconds and clustered nodes wait for cache
  rebalance. Roll one node at a time.

## Extension and migration quick reference

- In 24, `UserRepresentation.getAttributes()` contains custom attributes only;
  root profile fields remain dedicated properties. Use server-side
  `getRawAttributes()` only when a combined map is needed.
- In 25, token convenience setters are replaced by `exp`, `nbf`, and `iat`;
  `EnvironmentDependentProviderFactory.isSupported` takes `Config.Scope`.
- In 26.7, a `KeycloakSession` request transaction starts only once. Async REST
  work owns a separate session and transaction lifecycle.
- Audit Authorization Services resource URI templates before updating them:
  empty or slash-containing placeholders, invalid wildcard placement, and
  unmatched braces are rejected on create or update.
- Test custom themes against FreeMarker 2.3.32 behavior, particularly deprecated
  directives, undocumented syntax, and Java-internal access through `?api`.
