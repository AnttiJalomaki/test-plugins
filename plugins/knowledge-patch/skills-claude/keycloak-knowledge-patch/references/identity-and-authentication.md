# Identity and Authentication

## Client redirect, origin, and consent behavior

Valid redirect URIs are exact and case-sensitive unless configured with a
trailing wildcard. Even with that wildcard, a requested URI containing
userinfo or a `/../` parent-directory segment falls back to exact matching.
The full `*` pattern accepts every HTTP or HTTPS redirect and is unsafe for
production.

The client *Web Origins* values are embedded in its access token so the
application can decide whether to permit CORS requests. Only Keycloak client
adapters implement this behavior; do not treat it as portable OIDC client
metadata.

When *Consent required* is disabled, *Display client on screen* independently
controls whether the consent screen includes a client-specific item beside
client-scope consents. Custom client consent text is used only when consent and
that client item are both enabled.

## Logout

A backchannel logout URL is used only when front-channel logout is disabled.
When no backchannel URL exists, Keycloak can call the *Admin URL* using its
nonstandard adapter protocol. Only the legacy Keycloak Java OIDC adapters and
the Elytron WildFly OIDC adapter support that fallback. If neither URL exists,
no logout request is sent.

Enabling *Logout confirmation* shows a completion page after browser logout.
When the client supplies a validated `post_logout_redirect_uri`, the page
offers it as a continuation link or button rather than redirecting
automatically.

Keycloak 26 removes legacy logout `redirect_uri` handling and the
`legacy-logout-redirect-uri` and `suppress-logout-confirmation-screen` SPI
options. Use standards-based OIDC RP-Initiated Logout.

## Conditional 2FA and passkeys

The default Browser flow's conditional 2FA branch combines
*Condition - User Configured* and *Condition - credential*. The credential
condition skips the branch when passwordless WebAuthn has already authenticated
the user. WebAuthn and recovery-code executions are disabled by default; set
them to *Alternative* to let configured users choose them through
*Try Another Way*.

WebAuthn and WebAuthn Passwordless policies accept the specification values
`required`, `preferred`, and `discouraged` for *Discoverable credential*
(26.7.0). The former Boolean *Require Discoverable Credential* setting is
deprecated.

An authentication execution can carry a reference value. The
*Authentication Method Reference (AMR)* protocol mapper adds each successfully
completed execution's reference to the OIDC access- and ID-token `amr` claim.

## Flow selection, ACR, and LoA

Client Policies can use `AuthenticationFlowSelectorExecutor` to choose a flow
dynamically, set the resulting authentication level, and combine the choice
with conditions such as `ACRCondition`. An ACR-to-LoA mapping then exposes the
level in tokens.

Order *Conditional - Level Of Authentication* subflows from the lowest level to
the highest:

- The first conditional LoA subflow always runs on initial authentication.
- *Max Age* `0` makes that level valid only for the current authentication.
- An expired level that was not requested can reuse SSO but emits `acr=0`.
- An essential `claims` request must be satisfied or the request fails.
  `acr_values` is non-essential.
- Protect browser-carried requests with PAR or a request object and verify the
  returned `acr`.

```json
{"id_token":{"acr":{"essential":true,"values":["gold"]}}}
```

SAML service providers can request a particular authentication context class
for step-up authentication. The `step-up-authentication-saml` feature is
supported as of 26.7.0 rather than preview.

## Registration and credential reset entry points

Use `prompt=create` on an OIDC authorization request to open registration. The
`/registrations` authorization-path variant is deprecated. Replace `/auth`
with `/forgot-credentials` to start credential reset.

Do not link directly into `/login-actions` or `/broker`: those internal paths
bypass the OIDC or SAML flow and are unsupported.

## Session-limit authenticator

Add *User Session Count Limiter* as *Required* after the user has been
identified in Browser, Direct Grant, Reset Credentials, and Post Broker flows.
Keep one consistent configuration across those flows. It can reject the new
session or terminate the oldest; `0` disables the corresponding realm or
client limit.

In the Browser flow, put the limiter inside an alternative real-authentication
branch alongside the top-level Cookie execution. Normal SSO-cookie reuse then
avoids a second limit check. Session limiting is not available for CIBA.

## X.509 client authentication

X.509 client credentials require a *Certificate Authority subject DN* in
26.7.0 so the certificate is anchored to its intended CA. Older configurations
continue to run, but a future major version will reject create, update, or
import without this field.

Regex subject matching and HAProxy `ssl-cert-chain-prefix` are deprecated. Use
an exact subject match and `ssl-cert-chain`.

## DPoP restrictions

Keycloak 26.7.0 rejects implicit and hybrid authorization requests for clients
that require DPoP-bound tokens because those flows expose access tokens through
the front channel.

Sender-constrained token exchange has additional proof and client-identity
rules; see `protocols-token-exchange-and-brokering.md`.
