# Cryptography, TLS, certificates, SSH, and SFTP

## Post-quantum cryptography

### Algorithms and APIs

With an OpenSSL 3.5 build, 28.1 supports ML-DSA in `crypto:sign/4` and
`crypto:verify/5` through `mldsa44`, `mldsa65`, and `mldsa87`. ML-KEM is
available through `crypto:encapsulate_key/2` and
`crypto:decapsulate_key/3` with `mlkem512`, `mlkem768`, and `mlkem1024`.
The `public_key` and `ssl` applications integrate both families.

Release 28.3 adds the TLS 1.3 hybrid groups `x25519mlkem768`,
`secp384r1mlkem1024`, and `secp256r1mlkem768`. `public_key` and `ssl` also
add SLH-DSA support. Release 28.4 adds the SSH hybrid key exchange
`mlkem768x25519-sha256`, combining ML-KEM-768 with X25519.

In 29.0 these hybrids become preferred defaults. `ssl` places
`x25519mlkem768` first, and `ssh` places `mlkem768x25519-sha256` first, while
retaining fallback for unsupported peers. With a suitable certificate,
`ssl` prefers ML-DSA signatures followed by `slh_dsa_sha2_256f`.

### Unsupported backend behavior

Since 29.0.3, when the crypto backend lacks EdDH or EdDSA support,
`crypto:compute_key/4` and `crypto:generate_key/2,3` raise the structured
exception `error:{notsup, Info, Description}`:

```erlang
try crypto:generate_key(eddh, x25519) of
    Key -> Key
catch
    error:{notsup, Info, Description} ->
        {unsupported, Info, Description}
end.
```

## TLS configuration and protocol handling

### Fail-fast PEM validation

Since 28.1, TLS servers fail earlier when a configured PEM file is missing or
invalid. Treat bad paths or contents as deployment or startup failures.

### Duplicate and malformed handshake messages

`ssl` 11.4.2 in 28.2 rejects duplicate `change_cipher_spec` messages and a
second certificate message with an unexpected-message alert. Negative tests
and malformed peers that previously left corrupted handshake state now fail
immediately. When applying only the `ssl` patch, install its matching
`crypto` and `public_key` dependencies.

In 29.0.1, an SSL server sends an alert for bad-client cases that previously
closed silently.

TLS hardening in 29.0.3 also:

- rejects application data injected during the handshake plaintext window;
- rejects a second HelloRetryRequest;
- returns an `illegal_parameter` alert for PSK binder/identity mismatches;
- always validates TLS 1.3 stateless tickets against server lifetime and
  freshness data, even when the client reports an age of zero.

## Certificate validation

### SAN-only hostname matching

Since 29.0.1, `public_key` follows RFC 9525 and does not fall back to the
subject common name for hostname validation (CVE-2026-42790). A certificate
must contain a matching subject alternative name. Code matching `ssl` or
`public_key` errors must account for separate subject-name and
subject-alternative-name constraint errors.

For an individual application update, `ssl-11.7.1` requires
`public_key-1.21.1`.

### Path and OCSP hardening

Release 29.0.1 rejects expired OCSP responder certificates and corrects
RFC 5280 basic-constraint validation in certificate paths
(CVE-2026-42789).

Since 29.0.3, OCSP responses larger than 100 KB are rejected before ASN.1
decoding. Keep responder output and validation fixtures within this limit.

## SSH daemon behavior

Since 29.0, `ssh:daemon/2` does not enable shell, exec, or SFTP by default.
Opt in to each required service:

```erlang
ssh:daemon(Port, [
    {shell, {shell, start, []}},
    {exec, erlang_eval},
    {subsystems, [ssh_sftpd:subsystem_spec([])]}
    | Options
]).
```

Release 29.0.3 removes the obsolete SHA-1 authentication workaround for
OpenSSH 7.x. Retest legacy peers that depended on it.

## SFTP confinement

In 29.0.2, SFTP `READLINK` no longer exposes host paths outside the configured
server root; returned paths remain relative to that root.

In 29.0.3, `REALPATH` requests containing `..` can no longer reveal whether
paths outside the root exist. The server also caps each read request at
255 KiB, so clients must split larger reads.

