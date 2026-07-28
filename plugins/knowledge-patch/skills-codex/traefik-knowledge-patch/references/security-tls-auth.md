# Security, TLS, and authentication

Use this reference for ACME resolvers, certificate and upstream TLS policy,
ForwardAuth, forwarded headers, client-IP interpretation, and security-sensitive
middleware.

## Configure ACME accounts and trust per resolver

Certificate resolvers can use separate ACME account email addresses (since
3.2.0), so one resolver no longer forces its email onto the others. Each
resolver can also trust custom CA certificates, allowing a private or otherwise
non-default ACME CA to be scoped to that resolver.

ACME `certificatesDuration` supports a 30-day duration choice (since 3.2.0) for
certificates issued with that lifetime. Match the setting to CA policy; it does
not make a CA issue an unsupported lifetime.

## Tune ACME issuance

- Propagation-check controls can adjust challenge validation (since 3.3.0).
- `acme.profile` selects a certificate profile, and `acme.emailAddresses`
  supplies multiple contact addresses (since 3.4.0).
- ACME-managed certificates can use OCSP stapling (since 3.5.0).
- `acme.httpChallenge.delay` delays HTTP challenge processing when propagation
  timing requires it (since 3.5.0).
- The ACME provider HTTP timeout is configurable for slow certificate-service
  endpoints (since 3.5.0).
- `CertificateTimeout` bounds certificate operations (since 3.7.0).

Change propagation, delay, provider timeout, and certificate timeout
independently; they guard different parts of issuance. Test renewal as well as
initial issuance when changing resolver settings.

## Set downstream TLS policy

TLS configuration can disable session tickets (since 3.4.0). Disable them when
ticket-based resumption conflicts with the deployment's key-rotation or session
policy.

The `X25519MLKEM768` post-quantum-secure curve is supported since 3.5.0. Add it
only as part of an intentional curve-compatibility policy and verify the client
population.

Gateway API v1.5.1 listeners can specify multiple `certificateRefs` for SNI
selection (since 3.7.0). Patched 3.7 releases also:

- isolate TLS options for the same host on different entry points;
- apply SNI checks to routers that do not contain host rules;
- select deterministically among certificates that share a SAN;
- accept fragmented TLS ClientHello messages; and
- keep unrelated configuration active when a `TLSStore` refers to a missing
  Secret.

Exercise each affected entry point and SNI name. A successful handshake on one
entry point does not prove that TLS option isolation is correct on another.

## Secure upstream TLS

`BackendTLSPolicy` secures Gateway API backends and respects backend protocol
selection (since 3.2.0). With Gateway API v1.5.1,
`BackendTLSPolicy.caCertificateRefs` may point to Secrets containing private CA
bundles (since 3.7.0). Kubernetes CRD service TLS can instead add root CAs from
ConfigMaps (since 3.4.0).

`ServersTransport` can restrict upstream cipher suites (since 3.7.0). Keep
downstream and upstream TLS policy separate: listener settings do not constrain
the connection Traefik makes to a backend.

## Report CSP without enforcement

The Headers middleware can emit `Content-Security-Policy-Report-Only` (since
3.1.0). Use it to observe violations before enforcement, but protect the report
endpoint and do not mistake report-only delivery for an enforced policy.

## Normalize client IP decisions

`ipStrategy` supports an IPv6 subnet setting (since 3.2.0). Use it to normalize
IPv6 client addresses by subnet before an IP-based middleware decision, with a
prefix appropriate to the trust and privacy model.

A global option can stop Traefik from appending to `X-Forwarded-For` (since
3.7.0). Choose this according to which proxy owns the trusted forwarding chain;
disabling append changes what downstream services see as hop history.

The server can remove incoming header names containing underscores (since
3.7.0). Authentication middleware additionally drops untrusted underscore-
bearing `X-*` headers. Test the exact headers used for identity propagation so
legacy underscore names do not disappear unexpectedly.

## Configure ForwardAuth identity and redirects

ForwardAuth provides the following controls:

- `LogUserHeader` makes the authenticated user available to access logging
  (since 3.2.0). Treat it as identity-bearing log data.
- The authorization server's `Location` response header can be preserved (since
  3.3.0).
- `authSignInURL` provides a sign-in redirect target (since 3.7.0).
- `TrustForwardHeader` is deprecated (since 3.7.0); remove configurations that
  use it as the trust boundary for forwarded metadata.

Test success, denial, and sign-in redirect responses, including the destination
and status seen by the client.

## Bound and preserve authorization requests

ForwardAuth can send the incoming request body to the authorization service
(since 3.3.0). Authentication middleware warns when `maxBodySize` is absent
(since 3.6.0); set an explicit request-body ceiling before forwarding bodies.

ForwardAuth can preserve the original request method for the authorization
request (since 3.4.0). Enable this when authorization policy distinguishes
methods rather than expecting a normalized auth request.

`maxResponseBodySize` limits the authorization service's response (since
3.7.0). Bound both directions independently: `maxBodySize` protects the
forwarded request, while `maxResponseBodySize` protects response processing.

Patched 3.6 releases pass the correct `X-Forwarded-Port` to the authorization
service (since 3.6.21). Retest policies and redirect generation that distinguish
otherwise identical hosts by port.

## Limit headers and error-service disclosure

The server's maximum incoming request-header size is configurable (since
3.2.0). Set a deliberate ceiling that accommodates expected cookies and proxy
chains while rejecting excessive headers.

The Errors middleware can rewrite a handled response's status (since 3.4.0).
Its `errorRequestHeaders` option selects request headers sent to the error
service (since 3.7.0). Allow only what error rendering needs, especially for
authorization, cookie, and forwarded identity headers.

## Review plugin trust

Plugin manifests can enable unsafe Yaegi interpreter operations (since 3.5.0),
and plugins can use syscalls (since 3.6.0). These are explicit trust decisions,
not ordinary middleware configuration. Review plugin code and manifest changes,
and constrain the Traefik process before granting either capability.

## Apply patch-line security fixes

For deployments on the 3.7 line, 3.7.5 fixes CVE-2026-54761 and
CVE-2026-54762; 3.7.6 fixes CVE-2026-54763 through CVE-2026-54765; and 3.7.7
contains three further advisories. Use 3.7.7 rather than 3.7.0 when remaining
on that line.

## Validate security behavior

1. Exercise ACME account registration, challenge handling, issuance, renewal,
   and OCSP stapling for each resolver.
2. Test downstream SNI and certificate selection separately from upstream TLS
   verification and cipher policy.
3. Send ForwardAuth success, denial, redirect, oversized request, oversized
   response, method-sensitive, and port-sensitive cases.
4. Verify trusted and untrusted forwarded headers, IPv6 subnet normalization,
   underscore removal, and error-service header selection.
5. Inspect plugin capabilities and confirm the deployed patch line includes the
   required security fixes.
