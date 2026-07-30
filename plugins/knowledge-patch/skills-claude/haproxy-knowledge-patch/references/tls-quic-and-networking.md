# TLS, QUIC, and Networking

## Frontend certificates and policy

### Per-certificate TLS settings

The frontend `ssl-f-use` directive introduced in 3.2.0 references a certificate
from `crt-store` independently of `bind`. It can attach certificate-specific
TLS versions, ALPN, ciphers, and signature algorithms without an external
crt-list:

```haproxy
crt-store my_files
    load crt "foo.com.crt" key "foo.com.key" alias "foo"

frontend mysite
    bind :443 ssl
    ssl-f-use crt "@my_files/foo" ssl-min-ver TLSv1.2
```

Since 3.3.0, combining `strict-sni` and `default-crt` on a frontend `bind`
warns. `ssl_fc_crtname` returns the name of the certificate selected on an
incoming TLS connection (since 3.4.0).

### Protected keys and ECH

Global `ssl-passphrase-cmd` names an executable that prints the passphrase for
a protected private key (since 3.3.0). HAProxy tries passphrases it has already
retrieved before invoking the program again:

```haproxy
global
    ssl-passphrase-cmd /usr/local/bin/tls-key-passphrase
```

The `ech` argument on a TLS `bind` enables experimental Encrypted Client Hello
in 3.3.0. It requires `expose-experimental-directives`, and clients need the
corresponding public key from DNS.

## Automatic backend SNI

For server-side TLS, HAProxy 3.3.0 derives SNI automatically from the HTTP
`host` header. Use `sni-auto` or `no-sni-auto` for traffic and
`check-sni-auto` or `no-check-sni-auto` for health checks. Make a choice
explicit where a backend's certificate routing must not follow the request
host.

## ACME automation

### HTTP-01

The built-in ACME implementation introduced in 3.2.0 is experimental and
intended for one load balancer. Enable `expose-experimental-directives`; define
the directory, account, `HTTP-01` challenge, and virtual challenge map in an
`acme` section; then associate a `crt-store` load and its domains with the
account:

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
`acme renew @my_files/example` starts issuance and `acme status` lists tasks.
In the original HTTP-01 workflow, the issued certificate exists only in
running memory until `dump ssl cert @my_files/example` is saved to a file.

### DNS-01 and persistence

Since 3.3.0, HAProxy Data Plane API 3.3 can perform DNS-01 challenges, talk to
the DNS provider, and save issued certificates to the load balancer's
filesystem. The design still targets one load balancer; synchronize
certificates manually across multiple instances.

The `acme` trace source exposes automation events. The `haproxy-dump-certs`
utility introduced in 3.3.0 writes certificates obtained through a stats or
master socket to disk.

## QUIC handshake policy

`quic-initial` rules introduced in 3.1.0 run before the QUIC handshake
completes. Actions are `reject`, `accept`, `dgram-drop` for a silent datagram
drop, and `send-retry`. They can reject abuse or filter sources before the
client establishes a connection:

```haproxy
quic-initial dgram-drop if { src 192.0.2.0/24 }
```

## QUIC pacing and memory

Selecting `quic-cc-algo` enables transmit pacing automatically since 3.2.0,
and `bbr` no longer requires experimental directives. Set
`tune.quic.disable-tx-pacing` to disable pacing.

QUIC upload streams can consume 90 percent of connection memory by default.
Adjust this with `tune.quic.frontend.stream-data-ratio`; use
`tune.quic.frontend.max-tx-mem` for an optional global transmit-buffer cap in
3.2.0-era configuration. `haproxy -vv` reports socket-owner and UDP GSO
support.

The naming and scope evolve in later releases:

- Prefer `tune.quic.fe.*` to deprecated `tune.quic.frontend.*` names.
- Backend tuning added in 3.3.0 under `tune.quic.be.*` covers congestion
  control, idle timeout, glitch threshold, streams, pacing, and UDP GSO.
- `tune.quic.mem.tx-max` caps process-wide QUIC transmit memory.
- `quic-cc-algo` is accepted on `server` as well as `bind` in 3.4.0, and the
  change was also backported to 3.3. Frontend and backend algorithms can differ.
- `tune.quic.fe.stream.max-total` caps requests over one connection. At the
  limit HAProxy sends HTTP/3 GOAWAY, waits for remaining transfers, and closes.
- Replace global `no-quic` with `tune.quic.listen on|off`.

## Backend HTTP/3 and QMux

### HTTP/3 over QUIC

Backend HTTP/3 is experimental since 3.3.0. Prefix the server address with
`quic4@`, enable experimental directives, and retain normal backend TLS
verification:

```haproxy
global
    expose-experimental-directives

backend webservers
    server web1 quic4@172.16.0.11:443 check ssl verify required ca-file /etc/haproxy/ssl/myca.pem
```

### QMux over TCP

Experimental QMux in 3.4.0 carries QUIC frames over an ordered reliable byte
stream, allowing HTTP/3 between HAProxy endpoints where UDP cannot be used. It
requires `expose-experimental-directives` and `alpn h3` on the applicable TCP
`bind` or `server`:

```haproxy
global
    expose-experimental-directives

backend webservers
    server web1 127.0.0.1:443 ssl verify none alpn h3,h2
```

## DNS address-family policy

Global `dns-accept-family` accepts `ipv4`, `ipv6`, or `auto` since 3.2.0. A
fixed value disables the other family process-wide. `auto` tests IPv6
connectivity at startup and every 30 seconds, enabling IPv6 resolution only
while the probe succeeds. The default changed to `auto` in 3.3.0.

## TCP and kernel controls

- `tcp-md5sig` on `bind` and `server` enables the TCP MD5 Signature Option for
  uses such as proxied BGP sessions (since 3.3.0).
- `cc` on `bind` or `server` selects the TCP congestion-control algorithm for
  that listener or upstream (since 3.3.0).
- Global `tune.notsent-lowat.client` and `tune.notsent-lowat.server` reduce
  kernel-side socket buffering and unacknowledged data to lower memory use
  (since 3.2.0).
- The default `linux-glibc` build target requires Linux 4.17 since 3.3.0 so it
  can support Kernel TLS.
