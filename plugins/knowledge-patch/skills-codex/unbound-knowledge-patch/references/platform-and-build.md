# Platform and build behavior

## Trust anchor content

The compiled default root-key content in `unbound-anchor` includes key 38696
(since 1.21.0). Inspect the compiled-in data with:

```sh
unbound-anchor -l
```

## OpenSSL and dnstap builds

OpenSSL 3 builds configured with `OPENSSL_NO_DEPRECATED` are supported (since
1.21.0). Dnstap and `unbound-dnstap-socket` can also link without OpenSSL.

## Reproducible timestamps

Build timestamps use `SOURCE_DATE_EPOCH` in preference to wall-clock time, and
`--help` documents the variable (since 1.23.0).

## Service-unit behavior

The contributed `unbound.service` and `unbound_portable.service` units order
startup after `network-online.target` (since 1.21.0). Locally installed copies
of those templates inherit that ordering when updated.

Generated copies of both service units allow `CAP_NET_ADMIN` (since 1.24.0),
which is required by features that perform network-administration operations.

## Windows behavior

Module startup initializes `module-config` on Windows, so configured processing
modules are not silently skipped (since 1.21.0).

Windows builds initialize OpenSSL without reading `openssl.cnf` (since
1.25.0). This prevents local OpenSSL configuration from becoming a
privilege-escalation path. Any deployment that relied on implicit
`openssl.cnf` behavior must configure the required behavior elsewhere.

## Additional operating systems

The response-IP/ipset integration supports BSD PF tables (since 1.21.0).
Unbound can be built for QNX (since 1.25.0).

## Contributed ECC-GOST12 support

RFC 9558 ECC-GOST12 support is supplied as `contrib/gost12.patch` (since
1.25.0). It replaces the older GOST integration for deployments that apply
the contributed patch.
