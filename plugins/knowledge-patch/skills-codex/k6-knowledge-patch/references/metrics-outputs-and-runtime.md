# Metrics, Outputs, and Runtime

## Export metrics with the current shapes

### Rate metrics

Exported Rate metrics use one counter labeled with values `zero` and `nonzero`
(since 1.3.0). Downstream consumers must aggregate or filter that labeled shape
instead of expecting separate or unlabeled values.

`K6_OTEL_SINGLE_COUNTER_FOR_RATE` was removed in v2 (2.0.0). Delete it from
environment and deployment configuration; the single-counter migration can no
longer be postponed.

### Gauge extrema in Cloud output

Cloud output v2 reports Gauge `min` and `max` in their correct fields (1.8.0).
Queries no longer return the peak as the floor or the floor as the peak. Remove
query-side swaps added to compensate for the earlier output.

### Native histograms

The `native-histograms` feature makes trend metrics use experimental native
histograms (since 2.1.0):

```sh
k6 run --features native-histograms script.js
```

Enabled features are also metric tags, so dashboards can separate experimental
and ordinary runs.

## Configure OpenTelemetry

The stable output name is `opentelemetry` (since 1.4.0). The old
`experimental-opentelemetry` name still works but is deprecated. Replace the
deprecated `K6_OTEL_EXPORTER_TYPE` with `K6_OTEL_EXPORTER_PROTOCOL`:

```sh
k6 run --out opentelemetry script.js
```

The OpenTelemetry HTTP exporter accepts Basic Auth (since 2.1.0) through
`K6_OTEL_HTTP_EXPORTER_USERNAME` and `K6_OTEL_HTTP_EXPORTER_PASSWORD`, or the
`username` and `password` output-config keys:

```sh
K6_OTEL_HTTP_EXPORTER_USERNAME=user \
K6_OTEL_HTTP_EXPORTER_PASSWORD=secret \
k6 run --out opentelemetry script.js
```

Do not log the password or commit it in a shared output configuration.

## Configure output TLS

Experimental OpenTelemetry and Prometheus outputs default to TLS 1.3 (1.2.0).
This was a minor-release breaking change because those outputs were
experimental.

The experimental Prometheus remote-write output accepts
`K6_PROMETHEUS_RW_TLS_MIN_VERSION` to set its TLS floor (since 1.6.0); the
default remains TLS 1.3:

```sh
K6_PROMETHEUS_RW_TLS_MIN_VERSION=1.3 \
k6 run script.js -o experimental-prometheus-rw
```

## Use the built-in dashboard

The web dashboard is included in the k6 binary in v2; the separate
xk6-dashboard extension is unnecessary (2.0.0):

```sh
k6 run --out=web-dashboard script.js
```

## Handle gRPC values and metadata

`k6/net/grpc` is stable (1.0.0). gRPC marshaling represents floating-point
`NaN` and `Infinity` as strings rather than `null` (since 1.2.0); existing
scripts do not need a workaround.

The gRPC module accepts the `authority` pseudo-header (since 1.2.0). Supply it
for services that route or validate calls by authority.

## Use WebSockets

### Stable import path

WebSockets are stable at `k6/websockets` (since 1.6.0). The API is unchanged,
but the experimental import is deprecated:

```javascript
import ws from 'k6/websockets';
```

### Close codes and reasons

The WebSocket API can send a close code and reason and exposes both on the
close event (since 1.5.0). This capability first appeared on the experimental
path; use that path on 1.5.0 and the stable path on 1.6.0 or newer:

```javascript
import ws from 'k6/websockets';

export default function () {
  const socket = ws.connect('ws://example.com', socket => {
    socket.on('close', event => {
      console.log(event.code, event.reason);
    });
  });
  socket.close(1000, 'Normal closure');
}
```

### Typed-array buffering

Sending a TypedArray through `k6/websockets` increments `bufferedAmount`
correctly (1.8.0). It no longer becomes negative as typed-array data is sent;
remove guards that merely clamped the old erroneous counter.

## Derive and verify cryptographic values

`k6/crypto` is stable (1.0.0). It supports PBKDF2 password-based key derivation
(since 1.6.0).

The jslib TOTP package generates and verifies RFC 6238 time-based one-time
passwords from a base32 secret (since 1.6.0):

```javascript
import { TOTP } from 'https://jslib.k6.io/totp/1.0.0/index.js';

const totp = new TOTP('GEZDGNBVGY3TQOJQGEZDGNBVGY3TQOJQ', 6);
const code = await totp.gen();
const valid = await totp.verify(code);
```

## Write preview assertions

The URL-hosted `k6-testing` preview library provides `expect()` and
Playwright-style matchers for protocol and browser tests (since 1.2.0). It is
usable, but matcher and coverage gaps may remain:

```javascript
import { expect } from 'https://jslib.k6.io/k6-testing/0.5.0/index.js';
import http from 'k6/http';

export default function () {
  expect(http.get('https://quickpizza.grafana.com/').status).toBe(200);
}
```

Pin the URL version and validate that the matchers used by a test are present.

## Parse richer console output

`console.log()` traverses nested arrays and objects without dropping functions
or classes (since 1.5.0). Functions and classes render as
`"[object Function]"`, and circular references render as `"[Circular]"`
instead of collapsing the entire value to `[object Object]`.

It also renders `ArrayBuffer` bytes and shows typed-array types, lengths, and
values, including when nested (since 1.6.0). Update snapshot or log parsers that
assumed opaque binary values or shallow objects.
