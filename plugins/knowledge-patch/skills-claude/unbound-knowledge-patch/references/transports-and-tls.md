# Transports and TLS

## DNS over QUIC

### Build and runtime activation

Since 1.22.0, Unbound can serve DNS over QUIC when built against libngtcp2 and
a QUIC-capable OpenSSL:

```sh
./configure --with-libngtcp2=path --with-ssl=path
```

Enable the listener and set its memory budget:

```conf
server:
    quic-port: 853
    quic-size: 8m
```

Statistics expose `num.query.quic` and `mem.quic`. A build without DoQ support
ignores configured QUIC ports and warns when `quic-port` is set.

### Initialization and confinement

Since 1.23.0, the QUIC SSL context is created before chroot and privilege
drop. A QUIC listening context is created only when one is needed.

### Automatic port activation

Since 1.24.0, listing HTTPS or QUIC ports in `interface-automatic-ports`
initializes the corresponding protocol rather than only opening a port.

## Upstream transport

### Per-forward-zone overrides

Since 1.22.0, `forward-tcp-upstream` and `forward-tls-upstream` override the
global `tcp-upstream` and `tls-upstream` settings for one forward zone:

```conf
server:
    tcp-upstream: no
    tls-upstream: no

forward-zone:
    name: "."
    forward-tcp-upstream: yes
    forward-tls-upstream: yes
```

### Name-bound TLS reuse

Since 1.25.0, an existing upstream TLS connection is reused only when its TLS
name matches the new destination. Sharing an IP address is not sufficient
when the destination names differ.

## Listener isolation

Since 1.23.0, DoT and DoH have separate SSL contexts so each can use its own
ALPN values. Unbound also avoids opening unencrypted channels alongside
encrypted channels on the same port.

## TLS protocol selection

Since 1.25.0, `tls-protocols` selects the supported TLS versions used at
runtime. The transient `tls-use-system-versions` and `--enable-system-tls`
controls were removed before release.

Unbound 1.24.0 disabled TLS 1.2, while 1.24.1 allowed it again. Deployments
that require TLS 1.2 should not remain on 1.24.0.

## Certificate-aware reloads

Since 1.25.0, reloads detect changed certificate files and rebuild TLS
contexts for DoT, DoH, DoQ, and outgoing DoT. Certificate renewal therefore
does not require a full restart.

`unbound-control fast_reload` handles changes to:

- `tls-service-key`
- `tls-service-pem`
- `tls-cert-bundle`

It also propagates `iter-scrub-ns`, `iter-scrub-cname`, and
`max-global-quota` changes.

## Control listener ports

Since 1.25.0, `control-interface` accepts `IP@port`, so each remote-control
listener can select its port directly:

```conf
remote-control:
    control-interface: 127.0.0.1@8953
```

## Protocol error and EOF behavior

Since 1.25.0, malformed error cases receive error replies rather than silence,
without reflecting query fragments. CHAOS-class queries do not echo incoming
EDNS extended RCODEs. A TCP client EOF cancels pending replies and closes the
connection.
