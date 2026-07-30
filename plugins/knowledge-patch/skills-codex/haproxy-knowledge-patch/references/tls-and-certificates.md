# TLS and Certificates

## Frontend certificate policy

### Attach TLS policy to a `crt-store` certificate

Frontend directive `ssl-f-use`, added in 3.2.0, references a certificate from a
`crt-store` independently of `bind`. It can attach per-certificate TLS
versions, ALPN, cipher suites, and signature algorithms without maintaining an
external crt-list.

```haproxy
crt-store my_files
    load crt "foo.com.crt" key "foo.com.key" alias "foo"

frontend mysite
    bind :443 ssl
    ssl-f-use crt "@my_files/foo" ssl-min-ver TLSv1.2
```

### Encrypted Client Hello

The experimental `ech` argument on a TLS `bind` in 3.3.0 enables Encrypted
Client Hello. It requires global `expose-experimental-directives`, and clients
must be able to retrieve the corresponding public key through DNS.

## Backend TLS

### Automatic SNI

HAProxy 3.3.0 derives backend TLS SNI from the HTTP `host` header automatically.
Use `sni-auto` and `no-sni-auto` to control it for traffic; use
`check-sni-auto` and `no-check-sni-auto` for health checks. Combining
`strict-sni` with `default-crt` on a frontend `bind` now produces a warning.

### Protected private keys

Global `ssl-passphrase-cmd` in 3.3.0 points to a script that returns the
passphrase for a protected TLS private key. HAProxy first retries passphrases
that it has already retrieved before invoking the script again.

```haproxy
global
    ssl-passphrase-cmd /usr/local/bin/tls-key-passphrase
```

## ACME certificate automation

### Built-in HTTP-01 flow

The built-in ACME implementation introduced experimentally in 3.2.0 is
designed for one load balancer and requires
`expose-experimental-directives`. An `acme` section defines the directory,
account, HTTP-01 challenge, and virtual challenge map. A `crt-store` load ties
a certificate and its domains to that account.

```haproxy
global
    expose-experimental-directives

acme letsencrypt-staging
    directory https://acme-staging-v02.api.letsencrypt.org/directory
    account-key /etc/haproxy/account.key
    contact admin@example.com
    challenge HTTP-01
    map virt@acme

crt-store my_files
    crt-base /etc/haproxy/
    key-base /etc/haproxy/
    load crt "example.com.pem" acme letsencrypt-staging domains "example.com" alias "example"
```

The HTTP frontend must serve `/.well-known/acme-challenge/` from the ACME map.
Use `acme renew @my_files/example` to start issuance and `acme status` to list
tasks. The issued certificate exists only in process memory until output from
`dump ssl cert @my_files/example` is saved to a file.

### DNS-01 flow

ACME gains DNS-01 challenges in 3.3.0 through HAProxy Data Plane API 3.3. The
API communicates with the DNS provider and saves issued certificates on the
load balancer filesystem. The design still targets a single load balancer;
operators of multiple instances must synchronize the certificates themselves.

### Trace certificate automation

The `acme` trace source added in 3.3.0 exposes certificate-automation events:

```haproxy
traces
    trace acme sink stdout level user event +any verbosity clean start now
```

The Runtime API `trace` command also has an `ssl` source for TLS events since
3.2.0.

## Runtime certificate management

### Add aliases to a crt-list

`add ssl crt-list` stops requiring a certificate filesystem path to match its
in-memory name in 3.3.0. This allows `crt-store` aliases to work with a
crt-list, but the caller must ensure that each supplied path or alias identifies
the intended certificate.

### Dump certificates to disk

The `haproxy-dump-certs` utility added in 3.3.0 writes certificates obtained
through the stats socket or master socket to the filesystem. This is useful for
persisting certificates that otherwise exist only in the running process.

### Identify the selected certificate

The `ssl_fc_crtname` fetch added in 3.4.0 returns the name of the certificate
selected for the incoming TLS connection.
