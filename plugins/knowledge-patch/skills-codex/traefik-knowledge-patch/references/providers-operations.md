# Providers and operations

Use this reference for provider connectivity and discovery, listener ownership,
plugin execution, server protocol limits, and patch-line maintenance.

## Let systemd own listeners

Traefik supports systemd socket activation for listening sockets (since 3.1.0)
and extends it to UDP routing (since 3.4.0). Define socket units with the
protocol and address expected by each entry point, then ensure the service is
started with the inherited descriptors. Test both process restart and socket
traffic so the unit relationship is verified rather than merely loaded.

## Authenticate and bound provider requests

- Docker and Swarm providers support HTTP Basic Authentication when connecting
  to a protected provider endpoint (since 3.2.0).
- HTTP-provider requests include a `Host` header (since 3.3.0), allowing a
  configuration endpoint behind host-based routing to select the right virtual
  host.
- The HTTP provider accepts `maxResponseBodySize` (since 3.7.0). Bound the
  downloaded dynamic configuration to limit memory exposure and reject an
  unexpectedly large provider response.
- Docker and Swarm providers automatically negotiate the Docker API version
  (since 3.7.0), avoiding a fixed client-version assumption.

Keep provider credentials out of labels and logs. Verify authentication,
virtual-host selection, size-limit rejection, and refresh behavior against the
real endpoint.

## Configure label-derived backends

Docker, ECS, Docker Swarm, Consul Catalog, and Nomad can define backend server
URLs directly through provider labels (since 3.4.0). Use a complete URL suited
to the target provider and verify that label precedence produces the intended
server URL.

HTTP services can also preserve the path already present in a configured
backend server URL (since 3.2.0). Enable that service behavior when the label's
URL path must survive proxying.

## Choose discovery and refresh behavior

- Nomad can watch catalog changes rather than poll them (since 3.2.0). Choose
  the event-driven mode when prompt refreshes are required, and test reconnect
  behavior after the watch is interrupted.
- ECS discovery supports IPv6 (since 3.6.0).
- Docker discovery can include containers that are not running (since 3.6.0).
  Filter deliberately if stopped-container metadata must not produce routes.
- Kubernetes Ingress can publish Services of type `ExternalName` through
  Traefik (since 3.6.0).
- Consul, Consul Catalog, and Nomad log their provider namespace during startup
  (since 3.6.0). Use that message to verify namespace selection in multi-tenant
  deployments.

## Fail closed on plugin loading

`AbortOnPluginFailure` makes startup abort when a plugin cannot be loaded (since
3.3.0). Enable it when silently continuing without required middleware would
weaken routing or security policy.

Plugin manifests can permit unsafe operations in the Yaegi interpreter (since
3.5.0), and plugins can use syscalls (since 3.6.0). Both capabilities widen the
code and operating-system surface available to a plugin. Before enabling one:

1. inspect the manifest and source;
2. verify why the capability is necessary;
3. constrain filesystem, process, network, and container privileges around the
   Traefik process; and
4. make plugin-load failure visible or fatal according to policy.

## Tune server protocol limits

The maximum incoming request-header size is configurable (since 3.2.0). Pick a
limit that admits legitimate cookies and forwarding chains while bounding
header-based resource use.

HTTP/2 servers expose configurable HPACK table sizes (since 3.6.0). Tune these
only with an understood peer and workload requirement; changes affect header
compression state and memory use.

Health-check paths are validated as path-only values (since 3.7.0). Replace any
absolute health-check URL with a path and configure the backend address in the
service server definition.

## Handle WebSocket compatibility

Traefik 3.3.0 has a WebSocket-upgrade issue. A deployment that must remain on
that release should disable HTTP/2 extended CONNECT when launching Traefik:

```sh
GODEBUG=http2xconnect=0 traefik
```

Patched 3.7 releases allow WebSocket upgrades with `h2c` backends. Exercise a
real upgrade and bidirectional traffic when changing either the Traefik patch
level or the backend protocol.

## Watch authentication body warnings

Authentication middleware warns when `maxBodySize` is not configured (since
3.6.0). Treat the startup/runtime warning as a request to choose an explicit
forwarded-body limit, especially when ForwardAuth sends request bodies to an
authorization service.

## Maintain the 3.7 patch line

The 3.7 line includes security and compatibility fixes after 3.7.0:

- 3.7.5 addresses CVE-2026-54761 and CVE-2026-54762.
- 3.7.6 addresses CVE-2026-54763 through CVE-2026-54765.
- 3.7.7 addresses three additional advisories and also sanitizes paths produced
  by `ReplacePathRegex`.

Deployments staying on the 3.7 line should upgrade to 3.7.7 rather than remain
on 3.7.0. After upgrading, retest TLS option isolation, SNI checks, certificate
selection, redirects, CORS, path rewriting, and WebSocket behavior because the
patch line also corrects those request-handling paths.

## Validate operations

1. Restart under the target service manager and verify inherited TCP and UDP
   sockets.
2. Force provider authentication, timeout, oversized-response, disconnect, and
   reconnect cases.
3. Compare discovered services with the intended labels and provider filters.
4. Deliberately fail a required plugin in a non-production environment and
   verify the configured startup policy.
5. Confirm request-header, HPACK, health-path, and authentication-body limits.
6. Run the patch-line routing and TLS regression cases before promoting the
   image.
