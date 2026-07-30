# Admin, Account, and Organization APIs

## Client Admin API v2

Enable experimental `client-admin-api:v2` for strictly validated, declarative
OIDC and SAML client management (26.7.0). It is available through REST, Java,
generated JavaScript, and CLI clients. The management interface also publishes
its OpenAPI document.

The Operator uses this API for `KeycloakOIDCClient` and `KeycloakSAMLClient`
custom resources.

### Filtering and projection

`GET /admin/api/{realmName}/clients/v2` accepts a `q` expression with this SCIM
subset:

- Comparison operators: `eq`, `ne`, `co`, `sw`, `ew`, and `pr`.
- Boolean operators: `and`, `or`, and `not`.
- Parentheses for grouping.

String comparisons are case-sensitive and string literals require double
quotes. Boolean literals are unquoted. Malformed expressions, unknown fields,
and unsupported ordering operators `gt`, `ge`, `lt`, and `le` return HTTP 400.

```bash
curl -G -H "Authorization: Bearer $TOKEN" \
  --data-urlencode 'q=protocol eq "openid-connect" and enabled eq true' \
  --data-urlencode 'fields=clientId,displayName' \
  https://keycloak.example/admin/api/myrealm/clients/v2
```

For collection fields such as `roles` and `redirectUris`, `eq`, `co`, `sw`,
and `ew` match when any element matches. `ne` means no element equals the
given value. Repeat a predicate with `and` when multiple different values must
all be present.

Protocol-specific fields are genuinely null for the other client protocol.
Select these with `eq null` or `not ... pr`. The `fields` parameter projects
the response only after filtering the complete representation.

## Account REST feature boundaries

The `ACCOUNT_API` gate controls sessions, credentials, UMA resources,
organizations, verifiable-credential resources, applications, and application
consents. These routes return 404 when the feature is unavailable.

The profile root, `supportedLocales`, `linked-accounts`, and groups are not
guarded by that feature check in this service.

### Profile

Profile GET requires `manage-account` or `view-profile`. It includes
user-profile metadata unless `userProfileMetadata=false`.

Profile POST requires `manage-account`, validates the data in the `ACCOUNT`
user-profile context, and returns 204 on success.

### Application discovery

`GET /applications?name=` requires `manage-account` or `view-applications`.
Its result is the union of clients found in:

- online sessions;
- offline sessions;
- existing consents; and
- clients configured to always display in the account console.

Bearer-only clients are excluded. `name` is a case-insensitive substring match
against the configured client name, not the client ID.

### Application consent

At `/applications/{clientId}/consent`:

- GET accepts `briefRepresentation`, which defaults to `true`, and returns 204
  when no consent exists.
- DELETE returns 204.
- POST and PUT are both upserts, not separate create and replacement
  operations.
- Granted-scope IDs must resolve to realm client scopes, or to the client
  itself when client consent is required.
- Parameterized client scopes are rejected with HTTP 400.

Reading consent requires `manage-account`, `view-consent`, or
`manage-consent`. Creating, changing, or deleting it requires
`manage-account` or `manage-consent`.

## Delegated organization administration

Keycloak 26.7.0 adds three coarse-grained realm roles:

| Realm role | Access |
|---|---|
| `manage-organizations` | Organization write access |
| `view-organizations` | Organization read access |
| `query-organizations` | Organization search access |

Viewing organization members additionally requires `view-users` or an
equivalent fine-grained permission. `manage-realm` retains unrestricted
organization access.

Organizations are also resources in fine-grained admin permissions. Grant
view or manage rights to particular organizations and filter member queries by
the caller's user-level visibility.

## Organization group role inheritance

Realm and client roles assigned to an organization group propagate to every
member's `realm_access` and `resource_access` token claims (26.7.0).

Enabling *Add group role mappings* on the OIDC or SAML
*Organization Group Membership* mapper also organizes those roles by
organization inside the `organization` claim.

## Organization response compatibility

In 26.7.0:

- Organization-member listing returns brief users by default; request
  `briefRepresentation=false` for full user records.
- Invitation filters `email`, `firstName`, and `lastName` are
  case-insensitive exact matches. `search` remains substring matching.
- Organization-group representations return empty or populated `realmRoles`
  and `clientRoles` collections instead of null.
- General user-by-ID queries no longer return service accounts.

## Realm and identity-provider queries

Realm search matches both the technical realm name and the human-readable
display name (26.7.0).

From 26, ordinary realm representations no longer embed identity providers;
exports still do. API consumers must query the dedicated identity-provider
instances endpoint and use its filtering and pagination.

An identity-provider alias becomes immutable after creation in 26.7.0.
Attempting to change it through Admin REST returns HTTP 400.

## User representation attributes

From 24, `UserRepresentation.getAttributes()` contains custom attributes only.
Root fields such as username, email, first and last name, and locale remain
dedicated properties inherited from `AbstractUserRepresentation`.

Server-side code can call `getRawAttributes()` when it needs a combined map.
That method is not part of the serialized representation payload.

## Authorization resource URI validation

Authorization Services validates resource URI templates on create and update
in 26.7.0:

- Placeholders must be nonempty and cannot contain `/`.
- A wildcard is valid only as trailing `/*` or in a valid suffix such as
  `/*.html`.
- Unmatched braces are invalid.

Existing malformed values remain stored until updated. Audit every resource's
`uris` before upgrading so later edits do not fail unexpectedly.
