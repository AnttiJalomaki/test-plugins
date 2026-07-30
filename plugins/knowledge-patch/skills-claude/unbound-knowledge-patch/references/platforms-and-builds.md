# Platforms and builds

## Windows

### Module initialization

Since 1.21.0, Windows module startup runs and initializes `module-config`.
Configured processing modules are therefore set up rather than silently
missed.

### OpenSSL configuration isolation

Since 1.25.0, Windows builds initialize OpenSSL without loading `openssl.cnf`.
This prevents local OpenSSL configuration from being used for privilege
escalation. A deployment that relied on the implicit file must configure the
required behavior elsewhere.

## BSD PF tables

Since 1.21.0, the ipset integration supports BSD PF tables. This enables the
response-ip/ipset workflow on PF-based systems.

## Service units

### Network-online ordering

Since 1.21.0, the contributed `unbound.service` and
`unbound_portable.service` units start after `network-online.target`.
Locally installed copies of these templates inherit the new ordering only
when updated.

### Network-administration capability

Since 1.24.0, generated `unbound.service` and `unbound_portable.service`
units allow `CAP_NET_ADMIN`.

## OpenSSL and dnstap linkage

Since 1.21.0, builds support OpenSSL 3 installations compiled with
`OPENSSL_NO_DEPRECATED`. Both dnstap and `unbound-dnstap-socket` can link
without OpenSSL.

## Reproducible timestamps

Since 1.23.0, builds prefer `SOURCE_DATE_EPOCH` to the actual time.
`--help` documents the variable.

## QNX

Since 1.25.0, Unbound can be built for QNX.
