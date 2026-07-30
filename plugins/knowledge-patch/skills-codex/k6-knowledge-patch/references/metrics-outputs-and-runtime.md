# Metrics, outputs, and runtime

## OpenTelemetry

In 1.2.0, the experimental OpenTelemetry output began defaulting to TLS 1.3.
This was a minor-release breaking change while the output was experimental.

In 1.4.0, the output became stable as `opentelemetry`:

```sh
k6 run --out opentelemetry script.js
```

The `experimental-opentelemetry` alias remained but was deprecated.
`K6_OTEL_EXPORTER_TYPE` was deprecated in favor of
`K6_OTEL_EXPORTER_PROTOCOL`.

In 2.0.0, `K6_OTEL_SINGLE_COUNTER_FOR_RATE` was removed, so deployment
configuration cannot postpone the single-counter Rate representation.

Since 2.1.0, the HTTP exporter accepts Basic Auth through
`K6_OTEL_HTTP_EXPORTER_USERNAME` and `K6_OTEL_HTTP_EXPORTER_PASSWORD`, or
`username` and `password` output-config keys:

```sh
K6_OTEL_HTTP_EXPORTER_USERNAME=user \
K6_OTEL_HTTP_EXPORTER_PASSWORD=secret \
k6 run --out opentelemetry script.js
```

## Prometheus remote write

The experimental Prometheus output also began defaulting to TLS 1.3 in 1.2.0.
Since 1.6.0, set its minimum with
`K6_PROMETHEUS_RW_TLS_MIN_VERSION`; the default remains TLS 1.3:

```sh
K6_PROMETHEUS_RW_TLS_MIN_VERSION=1.3 \
k6 run script.js -o experimental-prometheus-rw
```

## Metric representation and correctness

Since 1.3.0, exported Rate metrics are one counter with a label whose values
are `zero` and `nonzero`. Downstream consumers must query that labeled shape.

Since 1.8.0:

- Redirects emit browser request metrics only for the applicable redirect and
  no longer duplicate samples for all earlier redirects.
- Cloud output v2 places Gauge `min` and `max` in the correct fields; the peak
  is no longer returned as the floor or vice versa.

Since 1.0.0-rc2, end-of-test output includes threshold values even when they
are not configured in `summaryTrendStats`.

## JavaScript and TypeScript

Since 1.0.0-rc1, the runtime supports logical assignment operators and array
destructuring in exports:

```javascript
let name;
name ??= 'default';
export const [first, second] = [1, 2];
```

Since 1.0.0, k6 executes `.ts` files directly without a separate transpilation
step. `k6/browser`, `k6/net/grpc`, and `k6/crypto` became stable in that
release.

The preview `k6-testing` library became available in 1.2.0 with `expect()` and
Playwright-style matchers for protocol and browser tests. It was usable but
did not yet cover every matcher or case:

```javascript
import { expect } from 'https://jslib.k6.io/k6-testing/0.5.0/index.js';
```

## Crypto and authentication helpers

Web Crypto is stable through global `crypto` since 1.0.0-rc1:

```javascript
export default function () {
  console.log(crypto.randomUUID());
}
```

`k6/experimental/webcrypto` was deprecated and scheduled for removal in
1.1.0.

Since 1.6.0, the crypto module supports PBKDF2 password-based key derivation.
The jslib TOTP package can generate and verify RFC 6238 time-based one-time
passwords from a base32 secret:

```javascript
import { TOTP } from 'https://jslib.k6.io/totp/1.0.0/index.js';

const totp = new TOTP('GEZDGNBVGY3TQOJQGEZDGNBVGY3TQOJQ', 6);
const code = await totp.gen();
const valid = await totp.verify(code);
```

## gRPC

Since 1.2.0:

- gRPC marshals `NaN` and `Infinity` float values as their string
  representations rather than `null`; scripts need no change.
- The gRPC module supports the `authority` pseudo-header for services that
  require it.

## WebSockets

In 1.5.0, the experimental WebSockets module allowed `close(code, reason)` and
exposed both values on close events.

In 1.6.0, WebSockets became stable at `k6/websockets`; the API stayed the same
and `k6/experimental/websockets` was deprecated.

Since 1.8.0, sending a TypedArray increments `bufferedAmount` correctly,
preventing the value from becoming negative while data is transmitted.

## Console representation

Since 1.5.0, `console.log()` traverses nested arrays and objects. Functions and
classes render as `"[object Function]"`, and circular references as
`"[Circular]"`, rather than collapsing the whole value.

Since 1.6.0, logging displays `ArrayBuffer` bytes and typed-array types,
lengths, and values, including nested instances.
