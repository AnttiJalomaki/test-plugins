# Client, Theme, and Extension Migrations

## Themes and bundled frontend resources

For 24, move welcome themes that extend the built-in theme from PatternFly 3 to
5 and place overridden images under common resources. Change Account Console
themes from `parent=keycloak.v2` to `parent=keycloak.v3`. In `content.json`,
rename `content` to `children` and remove `id`, `icon`, and `componentName`.

For 26, replace shared paths below `node_modules/...` with
`vendor/patternfly-v3`, `vendor/patternfly-v4`, `vendor/patternfly-v5`, or
`vendor/rfc4648`. Bundle Alpine.js or jQuery yourself because the common theme
no longer supplies them.

FreeMarker configuration defaults move from 2.3.0 to 2.3.32 in 26.7. Test
themes that depend on deprecated directives, undocumented syntax, or
Java-internal access through `?api`.

## Keycloak JavaScript

At the 24 package-exports boundary, replace deep imports with `keycloak-js` and
`keycloak-js/authz`. In 26, stop loading Keycloak JavaScript from the server;
the UMD/global build is removed. Pass configuration explicitly, run the
library in a secure context, and await `login()`, `createLoginUrl()`, and
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

## Feature selection

From 24, do not put the same feature in enabled and disabled lists. An
unversioned enabled name selects the latest supported implementation; pin
`name:vN` when upgrades must not change it. In 26, former `account3`, `admin2`,
and `login2` names become versioned base features such as
`--features=login:v1`. Disable a feature with its unversioned base name.

## User Profile enablement and extensions

Version 24 removes `declarative-user-profile` and enables User Profile in every
realm. Realms that had it enabled migrate with unmanaged attributes off; realms
that had it disabled migrate with unmanaged attributes on to preserve
permissive behavior. New validation includes a three-character minimum
username and prohibited-character checks. Existing realms retain the previous
`verify-profile` required-action state, while new realms enable it.

Update the 24 User Profile SPI: rename `Attributes.getValues()` to `get()` and
`getFirstValue()` to `getFirst()`, move `isRootAttribute` to
`UserProfileUtil`, and remove `getReadable(boolean)`. Move declarative-profile
theme changes into `login-update-profile.ftl` and `register.ftl`; move
broker-first-login profile customization into `idp-review-user-profile.ftl`.

## OIDC token shapes

Version 25 attaches the new default `basic` client scope to existing and new
OIDC clients and uses it to supply `sub` and `auth_time`. If a realm already
contains a scope named `basic`, migration skips this step. `session_state`
leaves tokens but stays in token responses. `nonce` becomes ID-token-only and
is omitted on refresh. Attach the supplied `Session State (session_state)` and
`Nonce backwards compatible` mappers for clients that require the old shapes.

## User and identity-provider representations

From 24, `UserRepresentation.getAttributes()` contains custom attributes only.
Username, email, names, and locale remain dedicated inherited properties.
Server-side code may use `getRawAttributes()` for a combined map, but that map
is not part of the representation payload.

In 26, ordinary realm representations no longer embed identity providers; only
exports do. Query the dedicated identity-provider instances endpoint and use
its filtering and pagination. From 26.7, an identity-provider alias is
immutable after creation; Admin REST returns HTTP 400 when an update tries to
change it.

## Logout

Version 26 removes legacy logout `redirect_uri` handling and the
`legacy-logout-redirect-uri` and `suppress-logout-confirmation-screen` SPI
options. Use OIDC RP-Initiated Logout.

## Events and extension APIs

Version 24 replaces the temporary-lockout log with the
`USER_DISABLED_BY_TEMPORARY_LOCKOUT` success event. In 26, realm deletion no
longer emits group-removal events; handle `RealmRemovedEvent`. Replace
credential-specific password and TOTP events with `UPDATE_CREDENTIAL` and
`REMOVE_CREDENTIAL`.

In 25, replace removed token convenience methods `expiration`, `notBefore`,
and `issuedAt` with `exp`, `nbf`, and `iat`.
`EnvironmentDependentProviderFactory` implementations must call
`isSupported(Config.Scope)`. In the 26.7 test framework, rename
`*ConfigBuilder` classes consistently to `*Builder` and update builder methods
toward `attribute`, `realmRoles`, and plural collection setters.

A `KeycloakSession` transaction may be started only once in 26.7, although
nested transactions remain supported. Do not restart the request transaction.
Async REST endpoints close their initiating session immediately; asynchronous
work must own its session and transaction lifecycle and cannot assume an active
request context.

## Email verification and registration

In 24, the `send-verify-email` Admin API uses `email-verification.ftl` instead
of `executeActions.ftl` and accepts a `lifespan` override. Use
`execute-actions-email` with `VERIFY_EMAIL` to retain the older template flow.

In 26.7, self-registration with Verify Email collects the profile before
verification and defers password, OTP, or passkey setup until afterward. The
deprecated *Always set password on register form* switch restores the earlier
sequence.

## Organization response shapes

In 26.7, organization-member listing returns brief users by default; pass
`briefRepresentation=false` for complete records. Invitation `email`,
`firstName`, and `lastName` filters are case-insensitive exact matches, while
`search` remains substring-based. Organization-group representations return
empty or populated `realmRoles` and `clientRoles` rather than `null`.
General user-by-ID queries no longer return service accounts.

## X.509 client authentication

X.509 client credentials in 26.7 add required *Certificate Authority subject
DN* to anchor the client certificate to the intended CA. Existing
configurations continue to run, but a future major release will reject create,
update, or import without it. Replace regex subject comparison with exact
matching and replace HAProxy `ssl-cert-chain-prefix` with `ssl-cert-chain`.

## Authorization resource URIs

Authorization Services in 26.7 reject malformed URI templates on create or
update. Require nonempty, slash-free placeholders. Allow wildcards only as a
trailing `/*` or a valid suffix such as `/*.html`; reject unmatched braces.
Existing malformed values remain until updated, so audit every resource's
`uris` before upgrading.

## Generated secrets and AES keys

New 26.7 client secrets are always 86 characters; remove shorter downstream
storage limits. Newly generated `aes-generated` providers default to 256-bit
keys. Existing providers do not change. Rotate by adding a higher-priority
provider with a 32-byte key and retain it until old sessions expire.

## Removed and deprecated feature switches

Remove obsolete `token-exchange-external-internal:v2` in favor of standard
token exchange. Remove
`spi-user-sessions--infinispan--use-batches` and
`spi-user-sessions--infinispan--max-batch-size`.

The OIDC *Bearer only* switch and OAuth 1.0a Twitter broker are deprecated.
Represent service-only clients by enabling no grants, and move Twitter
brokering to a generic OAuth v2 provider.
