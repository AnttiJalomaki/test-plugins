# Identity, Brokering, and Organizations

## Redirect URI and origin boundaries

Match valid redirect URIs exactly and case-sensitively unless a trailing
wildcard is registered. Even with a wildcard, force exact matching when the
requested URI contains userinfo or a `/../` parent-directory segment. The full
`*` pattern accepts any HTTP or HTTPS redirect and is unsafe for production.

Keycloak embeds the client's *Web Origins* values in its access token so an
application can decide whether to allow CORS. Only Keycloak client adapters
implement this convention; do not treat it as portable OIDC client metadata.

## Consent-screen client item

When *Consent required* is off, *Display client on screen* controls whether a
client-specific item appears beside client-scope consents. Use custom client
consent text only when consent and that client item are both enabled.

## Front- and backchannel logout delivery

Use a backchannel logout URL only when front-channel logout is disabled. If no
backchannel URL exists, Keycloak may fall back to the *Admin URL* through its
nonstandard adapter protocol. Only legacy Keycloak Java OIDC adapters and the
Elytron WildFly OIDC adapter support those callbacks. Send no logout request
when neither URL is configured.

With *Logout confirmation* enabled, show a completion page after browser
logout. If the client supplies a validated `post_logout_redirect_uri`, offer it
as a continuation link or button instead of redirecting automatically.

## External token retrieval

Identity Brokering API v2 is disabled by default. Authorize retrieval per
confidential client with *Allow retrieve external tokens* and an
identity-provider allow list. Use `POST` and handle OAuth-style JSON responses.
V2 replaces V1's per-user broker roles. *Store token in session* gives automatic
expiry cleanup and faster access but does not persist the token across sessions.
V1 remains enabled by default but is deprecated. (26.7.0)

## SCIM provisioning

Enable preview `scim-api` because the default profile leaves it disabled. The
API manages realm users and groups with CRUD, PATCH, filtering, pagination,
Enterprise User extensions, and schema discovery. (26.7.0)

## Delegated organization administration

Grant `manage-organizations`, `view-organizations`, or `query-organizations`
for coarse-grained write, read, or search access. `manage-realm` continues to
grant full access. Viewing members additionally requires `view-users` or the
equivalent fine-grained permission.

Treat organizations as fine-grained admin-permission resources when access must
be scoped to particular organizations. Grant view or manage on the selected
resource and expect member queries to be filtered by user-level visibility.
(26.7.0)

## Organization roles in tokens

Realm and client roles assigned to an organization group propagate to every
member's `realm_access` and `resource_access` claims. Enable *Add group role
mappings* on the OIDC or SAML *Organization Group Membership* mapper to group
those roles per organization inside the `organization` claim. (26.7.0)

## Realm discovery

Realm search matches both the technical realm name and the human-readable
display name. Account for either field when interpreting search results.
(26.7.0)
