# Authentication and Token Exchange

## Browser flows, passkeys, and execution metadata

The default Browser flow's conditional 2FA branch combines *Condition - User
Configured* with *Condition - credential*. The credential condition skips 2FA
after passwordless WebAuthn has already authenticated the user. WebAuthn and
recovery-code executions are disabled by default; set them to *Alternative* to
let configured users choose them through *Try Another Way*.

Assign a reference to an authentication execution when tokens must record how
authentication completed. The *Authentication Method Reference (AMR)* protocol
mapper adds the reference of every successful execution to the OIDC access- and
ID-token `amr` claim.

WebAuthn and WebAuthn Passwordless policies accept the specification values
`required`, `preferred`, and `discouraged` for *Discoverable credential*. Migrate
away from the deprecated boolean *Require Discoverable Credential* option.

## Dynamic flow selection and assurance

Use the Client Policy `AuthenticationFlowSelectorExecutor` to choose an
authentication flow dynamically and set its authentication level. Combine it
with conditions such as `ACRCondition`, then map LoA to ACR so the resulting
level appears in the token.

Order *Conditional - Level Of Authentication* subflows from lowest to highest:
the first always runs on a user's initial authentication. *Max Age* `0` makes a
level valid only for that authentication. An expired, unrequested level may
reuse SSO while producing `acr=0`.

An essential OIDC `claims` request must be satisfied or the request fails;
`acr_values` is non-essential. Protect assurance requests carried through the
browser with PAR or a request object and verify the returned `acr`.

```json
{"id_token":{"acr":{"essential":true,"values":["gold"]}}}
```

SAML service providers may request a specific authentication context class for
step-up authentication. Enable the supported `step-up-authentication-saml`
feature; it is no longer a preview feature. (26.7.0)

## Registration, reset, and session limits

Send `prompt=create` in an OIDC authorization request to open registration. The
`/registrations` authorization-path variant is deprecated. Replace `/auth` with
`/forgot-credentials` to start credential reset. Do not link directly into
`/login-actions` or `/broker`, which bypass the OIDC or SAML flow.

Add *User Session Count Limiter* as *Required* after the user is known in
Browser, Direct Grant, Reset Credentials, and Post Broker flows. Keep one
consistent configuration. Choose either denial of the new session or
termination of the oldest; `0` disables the corresponding realm or client
limit. In Browser flows, place the limiter in an alternative real-authentication
branch beside the top-level Cookie execution so normal SSO-cookie reuse is not
checked again. CIBA does not support session limiting.

## Standard token exchange v2

`token-exchange-standard:v2` is enabled by default, but every requester must be
a confidential client, enable *Standard token exchange*, and authenticate at
the token endpoint. Public requesters are unsupported. V2 exchanges a
same-realm Keycloak access token for an access token, ID token, or conditionally
a refresh token. It does not support the RFC 8693 `resource` parameter.

```bash
curl -u requester-client:secret https://keycloak.example/realms/test/protocol/openid-connect/token \
  -d grant_type=urn:ietf:params:oauth:grant-type:token-exchange \
  -d subject_token="$SUBJECT_TOKEN" \
  -d subject_token_type=urn:ietf:params:oauth:token-type:access_token \
  -d requested_token_type=urn:ietf:params:oauth:token-type:access_token
```

The requested `scope` adds the requester's optional scopes to its default
scopes. Repeated `audience` values only filter already resolved audiences,
client roles, and role-bearing client scopes; they never add an audience.
Reject an exchange that requests an unavailable audience. Require the subject
token to name the requester in `aud`, except when that token was issued to the
same client. Apply `downscope-assertion-grant-enforcer` when requested scopes
must also be restricted to scopes granted to the subject token.

A DPoP- or mTLS-bound subject token may be exchanged only by the client to which
it was issued. Supply a valid DPoP proof or the matching client certificate.

## Refresh, session, and revocation behavior

Requesting
`requested_token_type=urn:ietf:params:oauth:token-type:refresh_token` returns
both access and refresh tokens only when *Allow refresh token in Standard Token
Exchange* is not `No`. *Same session* rejects transient and offline subject
sessions as well as `offline_access`. An exchange never creates a new user
session.

Revoking the original access token does not revoke an exchanged access token.
It does revoke refresh tokens exchanged from the original, the requester client
session, and downstream refresh-token exchange chains.

Legacy V1 remains a disabled-by-default deprecated preview for external-token
exchange and user impersonation. It requires fine-grained admin permissions V1;
FGAP V2 deliberately has no token-exchange permissions. Its external-to-internal
JWT path checks signature and expiry but not `aud`, so it can accept an ID token
issued to another client. Grant `token-exchange` permission only to explicitly
trusted clients.

## Delegation

Enable experimental `token-exchange-delegation` to add a `delegation`
parameterized scope. It verifies that the requester may act for the target user
before exchange, requires user consent, and reassesses authorization during
refresh so revoked impersonation rights take effect immediately. (26.7.0)

## Parameterized scopes

Experimental parameterized scopes validate captured values as `string`,
`integer`, `boolean`, `username`, or `custom`; validate `custom` values against
an administrator-defined regular expression. A parameter type is mandatory
when creating a parameterized scope while the feature is enabled. (26.7.0)

Replace `--features=dynamic-scopes` with
`--features=parameterized-scopes`. The old Java model names are deprecated, and
database attributes migrate from `is.dynamic.scope` and
`dynamic.scope.regexp` to their `parameterized` equivalents.

## Verifiable credentials and assertion grants

Experimental OID4VCI configuration is available in the admin UI with HAIP
conformance, per-user credential management, and user-initiated issuance from
the account console. Enable dedicated `client-auth-abca` for attestation-based
client authentication. The pre-authorized code grant is
`oid4vc-vci-preauth-code`. Set `vc.refresh_interval_in_seconds` independently
from credential lifetime; it defaults to the smaller of seven days or the
credential lifetime. (26.7.0)

Enable experimental `identity-assertion-jwt` for partial ID-JAG receiver
support. Keycloak can act as the receiving authorization server, accept a signed
identity assertion at the token endpoint, and issue an access token without
another login. It cannot perform the other ID-JAG roles, so do not design a
complete flow around it. (26.7.0)

## Policy decisions and shared signals

Enable experimental `authzen` to expose Keycloak as an AuthZEN Policy Decision
Point. Evaluate subject, resource, and action against configured authorization
policies through either the single or batch endpoint; both return permit or
deny decisions. (26.7.0)

Enable experimental `ssf` to transmit signed Security Event Tokens using CAEP
1.0 or RISC 1.0. Configure streams, subjects, event types, and push or poll
delivery through the admin console or REST API. A durable outbox and
cluster-aware retry processing preserve delivery across restarts. (26.7.0)

## DPoP authorization-flow restriction

Do not use implicit or hybrid authorization flows for a client that requires
DPoP-bound tokens. Such requests are rejected because the flows expose access
tokens through the front channel.
