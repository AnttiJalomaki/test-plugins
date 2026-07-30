# Configuration, Authentication, and Security

Use this reference for TLS defaults, authentication and authorization
backends, OAuth/OIDC and LDAP configuration, protected resources, and
cluster-wide limits.

## TLS, Certificates, and Secrets

Starting in batch `4.1.0`, a node with no explicitly configured CA certificate
falls back to the system CA certificate list when one is available.

Starting in 4.1.4, `default_password` and `ssl_options.password` are treated as
encrypted only when the value begins with `encrypted:`. A colon in an
otherwise plain or generated password is not an encryption marker.

The obsolete etcd TLS and `*.cacerts` removals are detailed in
[upgrades-and-deprecations.md](upgrades-and-deprecations.md).

## Authentication Backend Selection

HTTP API traffic can use an authentication chain distinct from messaging
protocol connections:

```ini
auth_backends.1 = ldap
auth_backends.2 = internal
http_dispatch.auth_backends.1 = http
```

The caching authentication and authorization backend can be invalidated
explicitly:

```shell
rabbitmqctl clear_auth_backend_cache
```

Starting in 4.1.4, a configured authentication or authorization backend from
a known but disabled plugin prevents the node from starting. Enable the
plugin or remove the backend configuration.

Refreshing AMQP 0-9-1 connection credentials in 4.3 clears the permission
cache and immediately revalidates consumer permissions. Passive queue and
exchange declarations now require `configure` permission, like regular
declarations.

## OAuth 2.0 and OIDC

The OAuth 2 plugin no longer provides defaults for several Azure Entra and
Auth0 values. Configure required provider values explicitly.

Scope aliases can be set in `rabbitmq.conf`:

```ini
auth_oauth2.scope_aliases.admin = tag:administrator configure:*/*
auth_oauth2.scope_aliases.developer = tag:management configure:*/* read:*/* write:*/*
```

From 4.1.1, scope patterns can interpolate supported variables such as
`{vhost}` and `{sub}`.

The plugin supports configurable OpenID discovery endpoints and complex JWT
layouts used by providers such as Keycloak.

AMQP 1.0 clients can renew a JWT without disconnecting. If no replacement
arrives before expiry, RabbitMQ closes the connection. Starting in 4.1.4, a
failed renewal immediately closes a Stream Protocol connection, and a renewed
token is checked again for access to the connection's current virtual host.

## LDAP and HTTP Authorization

The LDAP plugin's `in_group_nested` query performs case-insensitive matching
as of batch `4.0.6`.

LDAP queries, including multi-line queries, can be configured directly in
`rabbitmq.conf` in 4.3.

An HTTP authentication backend can disclose a custom denial reason to AMQP
clients by returning `deny <Reason>` when disclosure is explicitly enabled:

```ini
auth_http.authorization_failure_disclosure = true
```

Keep disclosure disabled unless clients should receive backend-supplied
authorization details.

## Protected Resources and API Surfaces

Virtual hosts can be marked as protected from deletion through metadata.

Plugins can mark queues and streams as protected so applications cannot
delete them.

Require authentication for the management API reference page:

```ini
management.require_auth_for_api_reference = true
```

In 4.2, the HTTP API honors the `protected` user tag and refuses to modify or
delete such users. CLI operations can still remove the tag or delete and
recreate the user:

```shell
rabbitmqctl set_user_tags "a-user" "protected"
```

In 4.3, federation link restarts and Shovel management `DELETE` operations
require the `policymaker` user tag.

## Resource and Feature Limits

Starting in 4.1.4, `cluster_exchange_limit` caps exchanges declared by
applications across the cluster, including protocol-standard predeclared
exchanges:

```ini
cluster_exchange_limit = 200
```

Every node must use the same value.

Disable the local-random exchange type when load balancing cannot preserve
locality:

```ini
exchange_types.local_random.enabled = false
```

Declarations of that type then fail.

RabbitMQ 4.2 lets administrators disable individual queue types. Clients
cannot declare new queues or streams of a disabled type.

On Linux, macOS, and BSD, the `rabbitmq-server` startup script recognizes
`RABBITMQ_MAX_OPEN_FILES` from 4.1.4. It can raise a low soft limit when the
hard limit is already sufficient; it does not replace operating-system
hard-limit configuration.

## Virtual-Host Metadata

Starting in 4.1.1, metadata for a new virtual host includes its default queue
type. Definition exports therefore agree across export mechanisms.
