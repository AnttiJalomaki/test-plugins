# Security and authentication

Use this reference for authentication chains, OAuth/OIDC, LDAP, TLS defaults,
authorization, and protected administrative resources.

## Contents

- [Authentication backend chains](#authentication-backend-chains)
- [OAuth 2 and OIDC](#oauth-2-and-oidc)
- [LDAP](#ldap)
- [TLS and secret values](#tls-and-secret-values)
- [Protected resources](#protected-resources)
- [Authorization changes](#authorization-changes)
- [MQTT authorization](#mqtt-authorization)
- [WebSocket origin and login protection](#websocket-origin-and-login-protection)

## Authentication backend chains

HTTP API access can use a backend chain independent from messaging protocols:

```ini
auth_backends.1 = ldap
auth_backends.2 = internal
http_dispatch.auth_backends.1 = http
```

Starting in 4.1.4, a node refuses to start when a configured authentication or
authorization backend belongs to a known but disabled plugin. It no longer
boots with unusable client authentication.

Explicitly invalidate the caching authentication and authorization backend:

```shell
rabbitmqctl clear_auth_backend_cache
```

## OAuth 2 and OIDC

### Token renewal

- AMQP 1.0 clients can replace a JWT before expiry without disconnecting. The
  connection closes when no replacement is supplied in time.
- Starting in 4.1.4, a failed JWT renewal immediately closes a Stream Protocol
  connection, and a renewed token is checked again for access to the current
  virtual host.

### Provider configuration

The OAuth 2 plugin no longer provides defaults for several Azure Entra and
Auth0 values. Configure required provider values explicitly.

Configurable OpenID discovery endpoints and support for more complex JWT
structures improve compatibility with deployments such as Keycloak.

### Scope aliases and variables

Declare reusable aliases in `rabbitmq.conf`:

```ini
auth_oauth2.scope_aliases.admin = tag:administrator configure:*/*
auth_oauth2.scope_aliases.developer = tag:management configure:*/* read:*/* write:*/*
```

Starting in 4.1.1, scope patterns can interpolate selected variables,
including `{vhost}` and `{sub}`.

## LDAP

- `in_group_nested` performs case-insensitive matching.
- LDAP queries can be configured directly in `rabbitmq.conf`, including
  multi-line queries.

## TLS and secret values

- When no CA certificate is configured explicitly, a node falls back to the
  system CA list when one is available.
- Removed `*.cacerts` settings do not remove or rename `cacertfile`.
- Starting in 4.1.4, encrypted `default_password` and
  `ssl_options.password` values must begin with `encrypted:`. A colon in an
  ordinary or generated password does not imply encryption.

## Protected resources

### Virtual hosts, queues, and streams

Virtual-host metadata can prevent deletion. Plugins can likewise mark queues
and streams as protected against application deletion.

### Users

The `protected` tag prevents the HTTP API from modifying or deleting a user.
CLI operations can still remove the tag or delete and recreate the user:

```shell
rabbitmqctl set_user_tags "a-user" "protected"
```

### HTTP API reference

Require authentication for the `/api` reference page:

```ini
management.require_auth_for_api_reference = true
```

## Authorization changes

### AMQP 0-9-1

Refreshing credentials clears cached permissions and immediately rechecks
consumer authorization. Passive queue and exchange declarations require
`configure`, matching active declarations.

### Management operations

Federation link restarts and Shovel management `DELETE` operations require the
`policymaker` tag.

### HTTP backend denial details

An HTTP authentication backend may return `deny <Reason>` and disclose that
reason to AMQP clients when explicitly enabled:

```ini
auth_http.authorization_failure_disclosure = true
```

Keep disclosure disabled unless clients should receive backend-supplied denial
details.

## MQTT authorization

By default, an authorization failure closes the MQTT connection. Keep the
connection open and return a protocol-level error with:

```ini
mqtt.disconnect_on_unauthorized = false
```

## WebSocket origin and login protection

Web MQTT enforces `login_timeout` and bounds decompressed frames before and
after authentication. Use `web_mqtt.allow_origins` and
`web_stomp.allow_origins` to validate browser origins.
