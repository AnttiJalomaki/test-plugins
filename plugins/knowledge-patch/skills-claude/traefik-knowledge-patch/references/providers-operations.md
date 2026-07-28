# Providers and operations

## Docker, Swarm, ECS, Nomad, and catalog providers

Docker and Swarm can use HTTP Basic Authentication for protected provider
endpoints from 3.2.0.

Nomad can watch catalog changes rather than poll from 3.2.0, giving an
event-driven refresh path.

Docker, ECS, Swarm, Consul Catalog, and Nomad can read complete backend server
URLs from provider labels as of 3.4.0.

In 3.6.0, ECS discovery supports IPv6 and Docker can discover containers that
are not running. Consul, Consul Catalog, and Nomad log their provider namespace
during startup, making namespace selection visible during diagnosis.

Docker and Swarm negotiate the Docker API version automatically in 3.7.0.

## HTTP provider

HTTP provider requests include a `Host` header from 3.3.0, allowing a
host-routed configuration endpoint to serve the request correctly.

The 3.7.0 `maxResponseBodySize` setting bounds the size of downloaded dynamic
configuration. Treat an exceeded limit as a provider failure and size it for
the largest legitimate document with controlled headroom.

## Systemd socket activation

Systemd can own Traefik's TCP listeners from 3.1.0. Define and bind them in the
socket unit, then start Traefik as the accepting service. UDP routing joins
socket activation in 3.4.0.

Socket activation changes restart behavior: the listener can remain owned by
systemd while the process is replaced.

## Request and HTTP/2 resource limits

The maximum incoming request-header size is configurable from 3.2.0. Set it
from measured application needs; an undersized value rejects legitimate
requests, while an excessive value weakens the memory bound.

HTTP/2 servers expose HPACK table-size controls from 3.6.0. Tune header
compression tables only when interoperability or measured memory behavior
requires it.

The 3.3.0 WebSocket workaround disables HTTP/2 extended CONNECT:

```sh
GODEBUG=http2xconnect=0 traefik
```

Later 3.7 patches restore WebSocket upgrades through `h2c` backends.

## Support and diagnostics

The API exposes a support-dump endpoint from 3.3.0. Treat the dump as
diagnostic material that may contain deployment details and control its
distribution.

The API and dashboard base path became configurable in 3.3.0, supporting
mounting below a non-default route.

The Web UI's automatic theme became the default in 3.4.0.

The 3.7.0 dashboard adds a certificate overview with certificate domains,
expiration, and HTTP/TCP router attachments. Service pages show server
weights, and the dashboard name is configurable.

## Server and provider behavior

HTTP services can preserve the configured backend server path from 3.2.0.

In 3.7.0, the server can remove incoming header names containing underscores.
A global setting can disable appending to `X-Forwarded-For`.

Health-check targets are validated and must be path-only rather than absolute
URLs in 3.7.0.

Provider routing precedence is configurable in 3.7.0. Set it where routes from
different providers can match the same request rather than relying on load or
discovery order.

## Operational warnings

Authentication middleware logs a warning from 3.6.0 when `maxBodySize` is not
set. Treat it as a prompt to bound bodies sent to an authorization service.

Use `AbortOnPluginFailure` from 3.3.0 when missing plugin behavior should
prevent startup.
