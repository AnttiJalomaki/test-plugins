# QUIC, HTTP, and Networking

## QUIC admission and transport

### Run rules during the initial handshake

`quic-initial` rules, available in 3.1.0, run before the client finishes its
QUIC handshake. Supported actions are `reject`, `accept`, `dgram-drop` for a
silent drop, and `send-retry`. Use them for early source filtering and abuse
control.

```haproxy
quic-initial dgram-drop if { src 192.0.2.0/24 }
```

### Pacing and memory controls

Selecting `quic-cc-algo` in 3.2.0 automatically enables transmit pacing, and
`bbr` no longer needs experimental directives. Global
`tune.quic.disable-tx-pacing` disables pacing.

An upload stream may consume 90% of connection memory by default. Adjust that
share with `tune.quic.frontend.stream-data-ratio`. The optional global
`tune.quic.frontend.max-tx-mem` caps transmit-buffer memory. `haproxy -vv`
reports socket-owner and UDP GSO support for troubleshooting.

In 3.4.0, `quic-cc-algo` is also valid on `server`, so frontend and backend
congestion-control policies can differ; this addition was backported to 3.3.
Global `tune.quic.fe.stream.max-total` caps lifetime requests on one incoming
QUIC connection. At the cap HAProxy sends an HTTP/3 GOAWAY and closes after
remaining transfers complete.

### Enable or disable QUIC globally

Global `no-quic` is replaced in 3.3.0 by `tune.quic.listen on|off`, which
controls QUIC for every frontend listener. Global names beginning with
`tune.quic.frontend` are deprecated in favor of `tune.quic.fe`.

## HTTP/3 and QMux backends

### Native QUIC backend transport

Experimental HTTP/3 backends in 3.3.0 prefix the server address with `quic4@`.
They require `expose-experimental-directives` and ordinary backend TLS
verification.

```haproxy
global
    expose-experimental-directives

backend webservers
    server web1 quic4@172.16.0.11:443 check ssl verify required ca-file /etc/haproxy/ssl/myca.pem
```

Backend QUIC settings live under `tune.quic.be.*`, covering congestion
control, idle timeouts, glitch thresholds, stream limits, pacing, and UDP GSO.
The process-wide transmit-memory cap is `tune.quic.mem.tx-max`.

### QMux over TCP

Experimental QMux in 3.4.0 carries QUIC frames over an ordered, reliable byte
stream. It enables HTTP/3 between HAProxy endpoints when UDP is unavailable.
Enable `expose-experimental-directives` and configure `alpn h3` on the relevant
TCP `bind` or `server` line.

```haproxy
global
    expose-experimental-directives

backend webservers
    server web1 127.0.0.1:443 ssl verify none alpn h3,h2
```

## HTTP overload and glitch controls

### Bound HTTP/2 work

The following controls are available in 3.4.0:

- `tune.h2.fe.max-frames-at-once` and `tune.h2.be.max-frames-at-once` cap the
  number of incoming frames processed as a batch.
- `tune.h2.fe.max-rst-at-once` independently limits RST_STREAM processing.
  Values from 1 through 10 mitigate reset attacks, but very low values may add
  latency to interactive and gRPC traffic.
- `tune.h2.fe.max-total-streams` recycles an incoming connection after its
  lifetime stream limit.
- `tune.streams-elasticity` reduces per-connection concurrency as the frontend
  approaches `maxconn`.
- `tune.h2.fe.max-concurrent-streams rq-load` adjusts advertised concurrency
  from run-queue load, while `min` sets its advertised floor.

Global `tune.h2.log-errors` chooses error logging at `stream` scope,
connection-only scope, or not at all. Its default is the most verbose `stream`
mode.

### Detect HTTP/1 glitches

In 3.4.0, glitch detection also covers the HTTP/1 mux. Set frontend and backend
thresholds with `tune.h1.fe.glitches-threshold` and
`tune.h1.be.glitches-threshold`. When threshold termination is active, HAProxy
starts a graceful close at 75% rather than waiting to reach the limit.

### Gate glitch killing on CPU usage

Global `tune.glitches.kill.cpu-usage` in 3.2.0 is a CPU percentage from 0 to
100 above which over-threshold connections are killed. Its default `0` kills
at the threshold regardless of CPU load. A nonzero value requires
`tune.h2.fe.glitches-threshold` or `tune.quic.frontend.glitches-threshold`.

## DNS address-family policy

Global `dns-accept-family`, introduced in 3.2.0, accepts `ipv4`, `ipv6`, or
`auto` and disables the unselected family process-wide. `auto` probes IPv6 at
startup and every 30 seconds, enabling IPv6 resolution only while connectivity
is available.

The directive defaults to `auto` starting in 3.3.0, so IPv4 is enabled and
IPv6 is conditional unless the configuration selects a family explicitly.

## TCP controls

### TCP MD5 signatures

The `tcp-md5sig` argument on `bind` and `server` lines in 3.3.0 enables the TCP
MD5 Signature Option, as commonly required by BGP peers whose sessions are
proxied through HAProxy.

### Per-endpoint congestion control

The `cc` argument on `bind` and `server` lines in 3.3.0 selects the TCP
congestion-control algorithm for that listener or upstream server.

### Kernel send-buffer pressure

Global `tune.notsent-lowat.client` and `tune.notsent-lowat.server`, added in
3.2.0, limit kernel-side socket buffering and unacknowledged data. Use them to
reduce connection memory where lower buffering is acceptable.
