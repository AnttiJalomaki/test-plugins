# Protocols, Token Exchange, and Brokering

## Standard token exchange V2

`token-exchange-standard:v2` is enabled by default. Each requesting client must
still be confidential, enable *Standard token exchange*, and authenticate at
the token endpoint. Public requesters are unsupported.

V2 accepts a same-realm Keycloak access token as the subject token and can
return an access token, an ID token, or conditionally a refresh token. It does
not implement the RFC 8693 `resource` parameter.

```bash
curl -u requester-client:secret \
  https://keycloak.example/realms/test/protocol/openid-connect/token \
  -d grant_type=urn:ietf:params:oauth:grant-type:token-exchange \
  -d subject_token="$SUBJECT_TOKEN" \
  -d subject_token_type=urn:ietf:params:oauth:token-type:access_token \
  -d requested_token_type=urn:ietf:params:oauth:token-type:access_token
```

### Scope and audience resolution

The `scope` parameter adds the requester's optional scopes to its default
scopes. Repeated `audience` values only filter audiences, client roles, and
role-bearing client scopes already resolved for the request. They never add an
audience, and an unavailable requested audience rejects the exchange.

The subject token must contain the requester in `aud`, unless that token was
originally issued to the requester itself. Apply
`downscope-assertion-grant-enforcer` when the requested scopes must also be a
subset of the subject token's granted scopes.

### Sender-constrained subject tokens

A DPoP- or mTLS-bound subject token can be exchanged only by the client to
which it was issued. The exchange request must carry a valid DPoP proof or the
matching client certificate.

### Refresh tokens, sessions, and revocation

Requesting
`requested_token_type=urn:ietf:params:oauth:token-type:refresh_token` returns
both access and refresh tokens only when *Allow refresh token in Standard Token
Exchange* is not `No`.

Its *Same session* mode rejects transient subject sessions, offline subject
sessions, and `offline_access`. An exchange never creates a new user session.

Revoking the original access token does not revoke an already exchanged access
token. It does revoke refresh tokens exchanged from that token, the requester
client session, and every downstream refresh-token exchange chain.

## Delegation

The experimental `token-exchange-delegation` feature adds a parameterized
`delegation` scope (26.7.0). It verifies that the requester may act for the
target user before exchange. Delegation requires user consent and is
re-evaluated during refresh, so revoked impersonation rights take effect
immediately.

## Legacy token exchange V1

V1 is a disabled-by-default, deprecated preview kept for external-token
exchange and user impersonation. It requires fine-grained admin permissions
V1; FGAP V2 deliberately has no token-exchange permission model.

For JWT subject tokens, the V1 external-to-internal route validates signature
and expiry but not `aud`. It can therefore accept an ID token issued to another
client. Restrict the `token-exchange` permission to explicitly trusted clients.

Remove the obsolete `token-exchange-external-internal:v2` switch in 26.7.0 and
use standard token exchange where its same-realm boundary fits.

## Identity Brokering API v2

The disabled-by-default Identity Brokering API v2 (26.7.0) authorizes retrieval
of external tokens per confidential client. Enable *Allow retrieve external
tokens* and configure an identity-provider allow list. V2 uses `POST` and
OAuth-style JSON responses, replacing V1's per-user broker roles.

*Store token in session* keeps the token across access during a session,
automatically removes it at expiry, and avoids slower persistent retrieval.
Without it, retrieval can persist across sessions instead. Choose based on
lifetime and persistence needs.

V1 remains enabled by default but is deprecated.

## OID4VCI issuance

Experimental OID4VCI configuration is available in the admin UI (26.7.0), with
HAIP conformance, per-user credential management, and user-initiated issuance
from the account console.

- Enable `client-auth-abca` for attestation-based client authentication.
- The pre-authorized code grant uses `oid4vc-vci-preauth-code`.
- `vc.refresh_interval_in_seconds` controls credential refresh independently
  from credential lifetime. Its default is the smaller of seven days and the
  credential lifetime.

## Identity Assertion JWT Grant receiver

Experimental partial ID-JAG support lets Keycloak be the receiving
authorization server. With `identity-assertion-jwt`, the token endpoint accepts
a signed identity assertion and can issue an access token without another
login (26.7.0).

Only the receiver role is implemented. Keycloak cannot yet provide every role
needed for a complete ID-JAG flow.

## SCIM API

The preview SCIM API manages realm users and groups with CRUD, PATCH,
filtering, pagination, Enterprise User extensions, and schema discovery
(26.7.0). It is outside the default feature profile; enable `scim-api`.

## Shared Signals Framework

The experimental `ssf` feature lets a realm transmit signed Security Event
Tokens using CAEP 1.0 or RISC 1.0 through push or poll delivery (26.7.0).
Manage streams, subjects, and event types through the admin console or REST
API.

Events pass through a durable outbox and cluster-aware retry processor so
delivery survives restarts.

## AuthZEN evaluation

The experimental `authzen` feature exposes Keycloak as an AuthZEN Policy
Decision Point (26.7.0). It evaluates subject, resource, and action against
configured authorization policies. Single and batch endpoints both return
permit-or-deny decisions.

## Parameterized scopes

Experimental parameterized scopes validate captured values as `string`,
`integer`, `boolean`, `username`, or `custom`; custom values are checked
against an administrator-defined regular expression. A parameter type is
mandatory when creating a parameterized scope while the feature is enabled.

In 26.7.0, replace `--features=dynamic-scopes` with
`--features=parameterized-scopes`. Old Java model names are deprecated.
Database attributes migrate automatically from `is.dynamic.scope` and
`dynamic.scope.regexp` to their `parameterized` equivalents.
