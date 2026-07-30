# Upgrades, Extensions, and Themes

## Custom themes

### PatternFly and account-theme migration

When moving to 24:

- Welcome themes that extend the built-in theme must migrate from PatternFly 3
  to PatternFly 5.
- Put overridden images under common resources.
- Account Console themes must change `parent=keycloak.v2` to
  `parent=keycloak.v3`.
- In `content.json`, rename `content` to `children` and remove `id`, `icon`,
  and `componentName`.

In 26, shared third-party resource paths move from `node_modules/...` to:

- `vendor/patternfly-v3`
- `vendor/patternfly-v4`
- `vendor/patternfly-v5`
- `vendor/rfc4648`

Alpine.js and jQuery are no longer supplied by the common theme.

### User Profile templates

For 24, move declarative-profile customizations into
`login-update-profile.ftl` and `register.ftl`. Put broker-first-login profile
customizations in the new `idp-review-user-profile.ftl`.

### FreeMarker behavior

Keycloak 26.7.0 raises FreeMarker configuration defaults from 2.3.0 to 2.3.32.
Test custom themes that use deprecated directives, undocumented syntax, or
Java-internal access through `?api`.

## Keycloak JavaScript

The package-exports boundary in 24 removes deep imports. Import from
`keycloak-js` and `keycloak-js/authz`.

In 26, the server no longer serves Keycloak JS and the UMD/global build is
removed. Install the package in the application and pass configuration
explicitly. The library requires a secure context. `login()`,
`createLoginUrl()`, and `createRegisterUrl()` are asynchronous and must be
awaited.

```javascript
import Keycloak from "keycloak-js";
const keycloak = new Keycloak({
  url: "https://sso.example.com",
  realm: "example",
  clientId: "web"
});
await keycloak.login();
```

## Feature selection and renamed switches

From 24, a feature cannot appear in both enabled and disabled lists. An
unversioned enabled name selects the latest supported version; pin `name:vN`
when an upgrade must not change the chosen implementation.

The old `account3`, `admin2`, and `login2` names become versioned base features
in 26. For example, enable `--features=login:v1`; disabling uses the
unversioned base name.

In 26.7.0:

- Replace `--features=dynamic-scopes` with
  `--features=parameterized-scopes`. Old Java model names are deprecated and
  database attributes migrate from `is.dynamic.scope` and
  `dynamic.scope.regexp` to the corresponding `parameterized` names.
- Remove `token-exchange-external-internal:v2`; use standard token exchange.
- Remove `spi-user-sessions--infinispan--use-batches` and
  `spi-user-sessions--infinispan--max-batch-size`.
- The OIDC *Bearer only* setting is deprecated. Configure a service-only client
  with no enabled grants.
- The OAuth 1.0a Twitter broker is deprecated. Migrate to a generic OAuth v2
  provider.

## User Profile migration

Keycloak 24 removes `declarative-user-profile` and enables User Profile for
every realm.

- A realm that previously enabled the feature migrates with unmanaged
  attributes disabled.
- A realm that previously left it disabled migrates with unmanaged attributes
  enabled to preserve permissive behavior.
- New default validation constrains core fields, including a minimum
  three-character username and prohibited-character checks.
- Existing realms retain their former `verify-profile` required-action state;
  new realms enable it.

The User Profile SPI in 24 changes these Java APIs:

| Old | Replacement |
|---|---|
| `Attributes.getValues()` | `Attributes.get()` |
| `Attributes.getFirstValue()` | `Attributes.getFirst()` |
| `Attributes.isRootAttribute` | `UserProfileUtil.isRootAttribute` |
| `Attributes.getReadable(boolean)` | Removed |

## Password-hashing transition

Keycloak 24 changes the default from PBKDF2-SHA256 to PBKDF2-SHA512 with
210,000 iterations. Passwords in realms without an explicit hashing policy are
rehashed at login.

Keycloak 25 makes Argon2 the non-FIPS default and changes the default garbage
collector from ParallelGC to G1GC. Expect another one-time rehash and temporary
database activity.

## Preserving user sessions

To retain online sessions originating on 24, first upgrade to 25 with the
preview `persistent-user-sessions` feature enabled.

Only sessions already backed by remote Infinispan or embedded-cache JDBC
persistence can migrate. Enabling the feature later cannot safely merge
persisted and non-persisted sessions.

Version 26 replaces JBoss Marshalling with incompatible Protostream and clears
all caches. A direct upgrade that skips the 25 persistence migration loses
sessions.

In 26, all sessions are persisted by default. The standard cache configuration
bounds each session cache at 10,000 entries with one owner; custom cache XML
should use equivalent bounds. The two-minute idle-time grace period is gone.

Revoked access tokens persist across embedded-cache restarts by default. Use
`spi-single-use-object-infinispan-persist-revoked-tokens` to opt out.

## OIDC token-shape compatibility

In 25, the new default `basic` client scope is attached to new and existing
OIDC clients and supplies `sub` and `auth_time`. Migration is skipped in any
realm that already has a client scope named `basic`.

`session_state` leaves tokens but remains in the token response. `nonce`
becomes ID-token-only and is omitted on refresh.

For older clients that require the former token shapes, attach the supplied
`Session State (session_state)` and `Nonce backwards compatible` protocol
mappers.

## Email verification and self-registration

In 24, the `send-verify-email` Admin API switches from `executeActions.ftl` to
`email-verification.ftl` and accepts a `lifespan` override. Use
`execute-actions-email` with `VERIFY_EMAIL` to preserve the earlier template
flow.

In 26.7.0, self-registration with Verify Email collects the profile first and
defers password, OTP, or passkey setup until verification finishes. The
deprecated *Always set password on register form* setting restores the former
sequence temporarily.

## Java extension API changes

In 25:

- Token convenience methods `expiration`, `notBefore`, and `issuedAt` are
  removed. Use `exp`, `nbf`, and `iat`.
- `EnvironmentDependentProviderFactory` implementations must use
  `isSupported(Config.Scope)`.

The 26.7.0 test framework consistently renames `*ConfigBuilder` classes to
`*Builder`. Builder methods move toward `attribute`, `realmRoles`, and plural
collection setters.

### Transaction and asynchronous REST lifecycle

In 26.7.0, a `KeycloakSession` transaction can be started only once. Nested
transactions remain supported, but an extension must not restart the request
transaction.

An asynchronous REST endpoint closes its initiating session immediately.
Asynchronous work must own its session and transaction lifecycle and cannot
assume an active request context.

## Event-listener migration

Keycloak 24 replaces the temporary-lockout log with the successful
`USER_DISABLED_BY_TEMPORARY_LOCKOUT` event.

In 26, deleting a realm no longer emits group-removal events. Extensions must
handle `RealmRemovedEvent`. New `UPDATE_CREDENTIAL` and `REMOVE_CREDENTIAL`
events supersede password- and TOTP-specific credential event types.

## LDAP migration

Realm-level LDAP connection-pool settings are ignored in 26 because pooling is
JVM-wide. Move them to the documented system properties.

In 26.7.0:

- Existing binary user-attribute mappers migrate to `base64`.
- New user-attribute mappers default to `auto` and can explicitly select
  `base64` or `uuid`.
- Existing group mappers retain base64 behavior.
- New group mappers enable UUID decoding.

## Removed bundled integrations and artifacts

Keycloak 25 stops bundling the Oracle JDBC driver and removes the legacy
LinkedIn OAuth provider. Install a compatible Oracle driver and use the
remaining LinkedIn OIDC provider.

Version 26 removes the GELF handler, adapter and miscellaneous BOMs,
`keycloak-test-helper`, and the JEE admin client. The Jakarta admin client
remains.

## Related operational migrations

For migration work that changes deployment configuration, also consult
`server-operator-and-observability.md`. It contains the complete constraints
for:

- truststore sources, hostname verification, strict-FIPS generated
  truststores, hostname v2, and reverse-proxy headers;
- runtime-only cache settings, persistent-cache behavior, and external
  Infinispan boundaries;
- datasource XA defaults, transaction recovery, large-table index SQL, and
  PostgreSQL asynchronous commit;
- management-interface metrics, outbound response limits, shutdown, container
  heap sizing, bootstrap administrator variables, and Operator defaults.

API representation, organization, logout, X.509, DPoP, and authorization URI
changes are detailed in their corresponding indexed references.
