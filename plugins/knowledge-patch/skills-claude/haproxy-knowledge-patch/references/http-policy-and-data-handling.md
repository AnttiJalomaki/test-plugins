# HTTP Policy and Data Handling

## Compression evolution

### Minimum body sizes

Request and response compression can skip bodies smaller than a configured
byte count since 3.2.0. The response-side form at introduction was:

```haproxy
filter compression
compression direction response
compression minsize-res 256
```

Use the corresponding request or response minimum-size setting for the
direction being compressed so tiny bodies do not pay compression overhead.

### Split request and response filters

Since 3.4.0, request and response compression have separate `filter comp-req`
and `filter comp-res` filters rather than one compression filter plus a
direction. `compression-direction` is deprecated. Current response form:

```haproxy
backend webservers
    filter comp-res
    compression algo gzip
    compression type text/html text/plain application/json
```

### Explicit ordering

`filter-sequence` introduced in 3.4.0 determines execution order independently
of declaration order. This allows compression and bandwidth-limiting filters
to be rearranged. Every declared filter omitted from the sequence is skipped,
so the directive can also temporarily disable a filter without deleting its
declaration.

## Request and response actions

`set-retries` in `tcp-request` and `http-request` rules selects a retry count at
runtime (since 3.1.0):

```haproxy
http-request set-retries 0 if METH_POST
```

`pause` delays request or response processing by a fixed millisecond duration
or a sample expression (since 3.2.0):

```haproxy
http-request pause 250
http-response pause 250
```

Use it for deliberate policy pacing, such as slowing a rate-limit offender,
and include the delay in timeout planning.

`retry-on` accepts HTTP 421 since 3.2.0, enabling retry after a request is sent
to a server that cannot serve the intended authority.

## HTTP parsing and headers

The existing backend directives `accept-unsafe-violations-in-http-request` and
`accept-unsafe-violations-in-http-response` also tolerate missing expected
WebSocket headers since 3.2.0. This is compatibility mode for unsafe peers, not
a reason to enable relaxed parsing generally.

`option httpchk` accepts a Host header directly since 3.1.0, eliminating fake
strings embedded in the legacy `httpchk` line.

Since 3.3.0, `http-send-name-header` cannot target `connection`,
`content-length`, `host`, or `transfer-encoding`; overwriting these would
produce an invalid request.

## HTTP/2 overload controls

The following global controls were introduced in 3.4.0:

- `tune.h2.fe.max-frames-at-once` caps frontend frames processed in one batch.
- `tune.h2.be.max-frames-at-once` does the same for backend connections.
- `tune.h2.fe.max-rst-at-once` separately limits RST_STREAM processing. Values
  from 1 to 10 mitigate RST attacks, but very low values can add latency to
  interactive or gRPC traffic.
- `tune.h2.fe.max-total-streams` imposes a lifetime stream count and recycles
  the incoming connection at the limit.
- `tune.streams-elasticity` progressively lowers per-connection stream
  concurrency as the frontend approaches `maxconn`.
- `tune.h2.fe.max-concurrent-streams` accepts `rq-load` for run-queue-based
  adjustment and `min` for the advertised concurrency floor.

Global `tune.h2.log-errors` controls error reporting independently: `stream`
logs stream- and connection-scope errors and is the most verbose default;
connection-only and disabled modes reduce the scope.

## HTTP/1 and protocol glitch handling

The HTTP/1 multiplexer participates in glitch detection since 3.4.0. Configure
frontend and backend thresholds with `tune.h1.fe.glitches-threshold` and
`tune.h1.be.glitches-threshold`. When threshold-based termination is active,
HAProxy begins graceful close at 75 percent of the limit instead of waiting for
the connection to reach it.

Global `tune.glitches.kill.cpu-usage` introduced in 3.2.0 sets the CPU
percentage above which a connection that exceeds its glitch threshold is
killed. It accepts 0 through 100. Default `0` kills at the threshold regardless
of CPU load; a nonzero value requires `tune.h2.fe.glitches-threshold` or
`tune.quic.frontend.glitches-threshold` in 3.2.0-era naming.

## Conditional diagnostic samples

The `when(condition)` converter returns its input unchanged when the condition
is true and no value otherwise (since 3.1.0). It can selectively log expensive
or noisy values such as `bs.debug_str` and `fs.debug_str`.

`last_entity` and `waiting_entity` identify the operation interrupted by a
timeout or error. They can also identify the last evaluated rule behind an
accept, redirect, or deny.

## Connection, TLS, and counter samples

Added in 3.2.0:

- `bc_reused` reports whether the transfer reused a backend connection.
- `req.ssl_cipherlist`, `req.ssl_keyshare_groups`, `req.ssl_sigalgs`, and
  `req.ssl_supported_groups` expose binary capabilities from TLS ClientHello.
- `sc_key(<ctr>)` returns the tracked-counter key for the selected counter.
- `table_clr_gpc(<idx>[,<table>])` clears a general-purpose counter and returns
  its previous value.
- `table_inc_gpc(<idx>[,<table>])` increments one and returns its new value.
- `accept_date` and `request_date` fall back to the session date when no stream
  exists, including an early TLS-handshake failure.

## Directional byte counts

The directional fetches added in 3.3.0 distinguish the two legs and directions:

| Fetch | Meaning |
| --- | --- |
| `req.bytes_in` | Bytes received from the client; alias of `bytes_in` |
| `req.bytes_out` | Bytes sent to the server |
| `res.bytes_in` | Bytes received from the server; alias of `bytes_out` |
| `res.bytes_out` | Bytes sent to the client |

Do not infer direction from the alias name alone; use the request/response form
when a log or rule must be unambiguous.

## Version, timeout, certificate, and thread fetches

Since 3.4.0, `req.ver` and `res.ver` consistently return `major.minor` for
HTTP/1, HTTP/2, and HTTP/3. `capture.req.ver` and `capture.res.ver` consistently
return `HTTP/major.minor`.

Also added in 3.4.0:

- `cur_connect_timeout`, `cur_queue_timeout`, and `cur_tarpit_timeout` return
  active stream timeouts in milliseconds.
- `fe_tarpit_timeout` returns the configured frontend tarpit timeout.
- `ssl_fc_crtname` returns the selected incoming certificate name.
- `tgroup` returns the zero-based position of the calling thread group.

## Binary and cryptographic converters

The 3.3.0 converter additions and extensions are:

- `base2` renders each input byte as eight binary digits.
- `le2dec` interprets little-endian binary chunks as unsigned decimal values.
- `aes_gcm_enc` and `aes_gcm_dec` accept an optional AAD argument for
  authenticated additional data.

The 3.4.0 additions are:

- `jwt_decrypt_cert`, `jwt_decrypt_secret`, and `jwt_decrypt_jwk` decrypt JWT
  input with a certificate, base64-encoded secret, or JSON Web Key.
- `aes_cbc_enc` and `aes_cbc_dec` transform raw bytes using AES-128, AES-192,
  or AES-256-CBC according to the bits argument.
- `fe_exists` reports whether its input names a configured frontend.

## Stats-page disclosure

The Stats page hides the HAProxy version by default since 3.4.0. Add
`stats show-version` only when exposing the version is acceptable.
