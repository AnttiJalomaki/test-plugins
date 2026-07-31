# Scripting, Security, and Protocols

## Direct TypeScript execution

### Run `.ts` tests without a separate build (since 1.0.0)

k6 executes TypeScript test entry files directly.

```typescript
import http from 'k6/http';

interface Target {
  url: string;
}

const target: Target = { url: 'https://quickpizza.grafana.com/' };

export default function () {
  http.get(target.url);
}
```

```sh
k6 run script.ts
```

## Stable and preview script modules

### Use the stable core modules (since 1.0.0)

`k6/browser`, `k6/net/grpc`, and `k6/crypto` are stable and production-ready.

### Evaluate the preview assertions library (since 1.2.0)

The URL-hosted `k6-testing` preview library supplies `expect()` and
Playwright-style matchers for protocol and browser tests. It is usable, but
some matchers and test coverage may still be absent.

```javascript
import { expect } from 'https://jslib.k6.io/k6-testing/0.5.0/index.js';
import http from 'k6/http';

export default function () {
  expect(http.get('https://quickpizza.grafana.com/').status).toBe(200);
}
```

## gRPC values and metadata

### Marshal special floating-point values (since 1.2.0)

gRPC float values `NaN` and `Infinity` are encoded with their string
representations instead of `null`. Existing scripts require no changes.

### Send the authority pseudo-header (since 1.2.0)

The gRPC module supports the `authority` pseudo-header for services that
require it.

## Structured and binary logging

### Preserve nested values (since 1.5.0)

`console.log()` traverses nested arrays and objects without dropping functions
or classes. Functions and classes render as `"[object Function]"`; circular
references are marked `"[Circular]"` rather than collapsing the entire value
to `[object Object]`.

### Render binary values (since 1.6.0)

`console.log()` displays `ArrayBuffer` byte contents and typed-array types,
lengths, and values, including binary values nested in other objects.

## WebSockets

### Send and inspect close details (since 1.5.0)

The then-experimental WebSockets module added a close code and reason to
`close()` and exposed both values on the close event:

```javascript
import ws from 'k6/experimental/websockets';

export default function () {
  const socket = ws.connect('ws://example.com', socket => {
    socket.on('close', event => {
      console.log(event.code, event.reason);
    });
  });
  socket.close(1000, 'Normal closure');
}
```

### Import the stable module (since 1.6.0)

WebSockets are stable at `k6/websockets`. The API did not change during the
move. `k6/experimental/websockets` is deprecated and scheduled for removal.

```javascript
import ws from 'k6/websockets';
```

### Track TypedArray buffering correctly (since 1.8.0)

Sending a TypedArray through `k6/websockets` increments `bufferedAmount`
correctly. The counter no longer becomes negative as typed-array data is sent.

## Cryptography and one-time passwords

### Derive keys with PBKDF2 (since 1.6.0)

The crypto module supports PBKDF2 password-based key derivation.

### Generate and verify TOTP codes (since 1.6.0)

The jslib TOTP package generates and verifies RFC 6238 time-based one-time
passwords from a base32 secret.

```javascript
import { TOTP } from 'https://jslib.k6.io/totp/1.0.0/index.js';

const totp = new TOTP('GEZDGNBVGY3TQOJQGEZDGNBVGY3TQOJQ', 6);
const code = await totp.gen();
const valid = await totp.verify(code);
```

## Secret sources

### Fetch from an HTTP endpoint (since 1.5.0)

Secret management can retrieve secrets from an HTTP endpoint, allowing a
custom service to supply test values. The 1.5 implementation is only a mock;
it is not a production-ready external secret-manager integration.

### Configure sources through the environment (since 1.7.0)

`K6_SECRET_SOURCE` accepts the same value syntax as `--secret-source`.

```sh
K6_SECRET_SOURCE='mock=cool="not cool secret"' k6 run script.js
```

Cloud secrets used with local Cloud execution are covered in
[Local Cloud execution](cli-cloud-and-configuration.md#local-cloud-execution).

## Execution status

### Detect an explicit failure (since 1.6.0)

Code consuming execution status can distinguish a test marked failed with
`exec.test.fail()` through `ExecutionStatusMarkedAsFailed`.
