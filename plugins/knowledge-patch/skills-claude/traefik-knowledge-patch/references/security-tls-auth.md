# Security, TLS, and authentication

## Patch-line security

Move production deployments beyond the first 3.7.0 build. Version 3.7.5 fixes
CVE-2026-54761 and CVE-2026-54762; 3.7.6 fixes CVE-2026-54763 through
CVE-2026-54765; and 3.7.7 fixes three additional advisories. Use at least
3.7.7 when remaining on this release line.

The later patch releases also correct TLS option isolation across entry points,
SNI checking on routers without host rules, deterministic selection when
certificates share a SAN, path sanitization, CORS behavior, and WebSocket
upgrades through `h2c` backends.

## ACME resolver isolation

Since 3.2.0, each certificate resolver may use a different account email and
different custom CA certificates. This permits separate public and private
issuers without sharing contact or trust configuration. A 30-day
`certificatesDuration` is supported for issuers that use that lifetime.

Propagation-check controls were added in 3.3.0 so a resolver can tune challenge
validation rather than depend on a single propagation behavior.

Traefik 3.4.0 adds `acme.profile` and `acme.emailAddresses`, allowing a
certificate profile and multiple contact addresses.

In 3.5.0, ACME certificates can use OCSP stapling. HTTP challenges add
`acme.httpChallenge.delay`, and the ACME provider HTTP timeout is configurable
for slow certificate endpoints.

The 3.7.0 ACME settings add `CertificateTimeout`, which bounds certificate
operations independently of the earlier provider and challenge controls.

## TLS configuration

Kubernetes CRD service TLS can add trusted roots from ConfigMaps (3.4.0).
Gateway API v1.5.1 `BackendTLSPolicy.caCertificateRefs` can use Secrets that
hold private CA bundles (3.7.0).

TLS session tickets can be disabled from 3.4.0. The `X25519MLKEM768`
post-quantum-secure curve is supported from 3.5.0.

`ServersTransport` can restrict upstream cipher suites in 3.7.0. Fragmented TLS
ClientHello messages are accepted, and a TLSStore whose Secret is missing no
longer takes unrelated configuration down.

Gateway listeners may supply multiple `certificateRefs` for SNI selection
(3.7.0). Later patches isolate TLS options for the same host on separate entry
points, enforce SNI on routers without host rules, and make certificate
selection deterministic when SANs overlap.

## ForwardAuth request and response controls

ForwardAuth added identity logging through `LogUserHeader` in 3.2.0. Use it
when the trusted authenticated-user header should be copied into access logs.

From 3.3.0, ForwardAuth can preserve the authorization server's `Location`
header and can send the incoming request body. Set request body limits when
enabling body forwarding.

The original request method can be preserved for the authorization request
from 3.4.0.

In 3.7.0, `authSignInURL` supports sign-in redirects and
`maxResponseBodySize` bounds the authorization response.
`ForwardAuth.TrustForwardHeader` is deprecated; move to an explicit trusted
proxy/header design.

Release 3.6.21 corrects the `X-Forwarded-Port` sent to the authorization
service.

Authentication middleware warns from 3.6.0 when `maxBodySize` is not
configured. In 3.7.0, it discards untrusted `X-*` headers whose names contain
underscores. The server can also remove incoming underscore-bearing header
names globally.

## External provider authentication

Docker and Swarm provider endpoints accept HTTP Basic Authentication from
3.2.0. Store credentials outside checked-in dynamic configuration and use TLS
when the provider endpoint crosses a trust boundary.

## Rate limits and shared state

The RateLimit middleware can keep state in Redis from 3.4.0, allowing a limit
to apply across multiple Traefik instances. Treat Redis availability,
partition behavior, and key isolation as part of the enforcement design.

Service-level middleware in 3.7.0 can attach the rate limit once to a backend
used by many routers.

## Header and error controls

Use report-only CSP from the Headers middleware to observe violations before
enforcement (`Content-Security-Policy-Report-Only`, 3.1.0).

The Errors middleware can rewrite HTTP status while serving an error response
from 3.4.0. Its 3.7.0 `errorRequestHeaders` setting limits which request
headers reach the error service.

A global 3.7.0 option can stop appending to `X-Forwarded-For`; use it only with
a deliberate upstream trust policy.

## Plugin trust and startup behavior

`AbortOnPluginFailure`, introduced in 3.3.0, makes startup fail when a plugin
cannot load instead of silently continuing without it.

Plugin manifests can opt into unsafe Yaegi operations from 3.5.0. Plugins can
also use system calls from 3.6.0. Either capability widens the plugin's access;
review the source and distribution chain as privileged application code.

Enable fail-closed startup when the plugin implements required security or
routing behavior.
