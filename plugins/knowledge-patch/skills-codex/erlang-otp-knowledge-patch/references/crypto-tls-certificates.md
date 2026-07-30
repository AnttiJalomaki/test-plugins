# Cryptography, TLS, and Certificates

## Post-quantum primitives

When OTP 28.1 is built with OpenSSL 3.5, `crypto:sign/4` and
`crypto:verify/5` support ML-DSA with the `mldsa44`, `mldsa65`, and `mldsa87`
algorithm atoms. ML-KEM is exposed through `crypto:encapsulate_key/2` and
`crypto:decapsulate_key/3` with `mlkem512`, `mlkem768`, and `mlkem1024`.
`public_key` and `ssl` integrate both families as well.

OTP 28.3 adds the TLS 1.3 hybrid groups `x25519mlkem768`,
`secp384r1mlkem1024`, and `secp256r1mlkem768`. The `public_key` and `ssl`
applications also add SLH-DSA support.

SSH supports the hybrid `mlkem768x25519-sha256` key exchange starting in OTP
28.4. In OTP 29.0, the hybrid algorithms become preferred defaults: `ssl`
puts `x25519mlkem768` first and `ssh` puts `mlkem768x25519-sha256` first, while
retaining fallback for peers without support. When a suitable certificate is
configured, `ssl` prefers ML-DSA signatures followed by
`slh_dsa_sha2_256f`.

Backend support remains runtime-dependent. In OTP 29.0.3,
`crypto:compute_key/4` and `crypto:generate_key/2,3` report a missing EdDH or
EdDSA implementation as structured exceptions:

```erlang
try crypto:generate_key(eddh, x25519) of
    Key -> Key
catch
    error:{notsup, Info, Description} ->
        {unsupported, Info, Description}
end.
```

Handle `{notsup, Info, Description}` rather than depending on the earlier
unstructured failure.

## Certificate configuration and validation

TLS servers on OTP 28.1 fail earlier when a configured PEM file is missing or
invalid. Treat paths and contents as startup validation failures rather than
expecting a later handshake error.

OTP 29.0.1 changes hostname verification to follow RFC 9525 and fixes
CVE-2026-42790. Validation no longer falls back to a certificate subject's
common name; the certificate must contain a matching subject alternative
name. Error handling must also account for distinct subject-name and
subject-alternative-name constraint errors.

When applying application patches rather than a full OTP patch, note that
`ssl-11.7.1` requires `public_key-1.21.1`.

The same OTP 29.0.1 patch rejects expired OCSP responder certificates and
corrects RFC 5280 basic-constraint validation for certificate paths
(CVE-2026-42789).

OTP 29.0.3 rejects an OCSP response larger than 100 KB before ASN.1 decoding.
Certificate-validation integrations must keep responder output within the
limit.

## TLS handshake hardening

In OTP 28.2, `ssl` 11.4.2 rejects duplicate `change_cipher_spec` messages and
a second certificate message with an unexpected-message alert. Malformed
peers and negative tests that previously left corrupted handshake state now
fail immediately. An individually patched `ssl` requires the corresponding
`crypto` and `public_key` versions.

OTP 29.0.1 sends a TLS alert for malformed-client cases that previously
terminated the connection silently. Protocol tests and clients should expect
an alert rather than only a disconnect.

OTP 29.0.3 tightens several TLS paths:

- clients reject application data injected during the handshake plaintext
  window;
- clients reject a second HelloRetryRequest;
- a PSK binder/identity mismatch produces an `illegal_parameter` alert; and
- TLS 1.3 stateless tickets are always checked against server lifetime and
  freshness data, even if the client reports an age of zero.

Update negative tests to assert these immediate failures and alerts.

## PKICMP support

`public_key` 1.19 in OTP 28.2 adds ASN.1 support for the Public-Key
Infrastructure Certificate Management Protocol (PKICMP). If installing that
application patch by itself, first satisfy its dependency on the
OpenSSL-backed `crypto` version shipped in OTP 28.1.

## Encrypted crash dumps

OTP 29.0 can be configured with `--enable-encrypted-crash-dumps`. Use this
runtime build option where crash dumps may contain secrets and the deployment
has an operational plan for its encrypted artifacts.
