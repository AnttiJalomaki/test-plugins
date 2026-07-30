# Admin and Account APIs

## Client Admin API v2

Enable experimental `client-admin-api:v2` for strictly validated, declarative
OIDC and SAML client management. Use it through REST, Java, generated
JavaScript, or CLI clients. Its OpenAPI specification is exposed on the
management interface. The Operator uses this API for `KeycloakOIDCClient` and
`KeycloakSAMLClient` custom resources. (26.7.0)

`GET /admin/api/{realmName}/clients/v2` accepts a `q` filter with the SCIM
subset `eq`, `ne`, `co`, `sw`, `ew`, and `pr`, combined using `and`, `or`,
`not`, and parentheses. String comparisons are case-sensitive and require
double quotes; booleans are unquoted. Malformed filters, unknown fields, and
unsupported `gt`, `ge`, `lt`, or `le` operators return HTTP 400.

```bash
curl -G -H "Authorization: Bearer $TOKEN" \
  --data-urlencode 'q=protocol eq "openid-connect" and enabled eq true' \
  --data-urlencode 'fields=clientId,displayName' \
  https://keycloak.example/admin/api/myrealm/clients/v2
```

For collection fields such as `roles` and `redirectUris`, `eq`, `co`, `sw`,
and `ew` match when any element matches. `ne` means no element equals the
value. Repeat a condition with `and` to require multiple values.
Protocol-specific fields are genuinely null on the other client type; select
them with `eq null` or `not ... pr`. Apply `fields` projection only after
filtering the complete representation.

## Account REST feature boundaries

When the `ACCOUNT_API` feature gate is disabled, return 404 for sessions,
credentials, UMA resources, organizations, verifiable-credential resources,
applications, and application-consent operations. The profile root,
`supportedLocales`, `linked-accounts`, and groups are not guarded by that check
in this service.

Profile GET requires `manage-account` or `view-profile` and includes
user-profile metadata unless `userProfileMetadata=false`. Profile POST requires
`manage-account`, validates in the `ACCOUNT` user-profile context, and returns
204 on success.

## Application discovery

`GET /applications?name=` requires `manage-account` or `view-applications`.
Return the union of clients used by online sessions, offline sessions, existing
consents, and clients configured to always display in the account console.
Exclude bearer-only clients. Match `name` as a case-insensitive substring of
the configured client name, not the client ID.

## Consent CRUD

At `/applications/{clientId}/consent`, GET accepts `briefRepresentation`
(default `true`) and returns 204 when no consent exists. DELETE also returns
204. Treat both POST and PUT as upserts, not distinct create and replace
operations.

Resolve granted-scope IDs to realm client scopes, or to the client itself when
its consent is required. Reject parameterized client scopes with HTTP 400.
Reading consent requires `manage-account`, `view-consent`, or `manage-consent`;
changing or deleting it requires `manage-account` or `manage-consent`.
