# Scripting, Security, and Protocols

## TypeScript and JavaScript runtime behavior

Direct `.ts` execution became available in `1.0.0` without a separate
transpilation step:

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

The JavaScript runtime gained logical assignment and destructuring in exports
in `1.0.0-rc1`:

```javascript
let name;
name ??= 'default';
export const [first, second] = [1, 2];
```

`k6/browser`, `k6/net/grpc`, and `k6/crypto` became stable and
production-ready in `1.0.0`.

## Crypto and authentication helpers

Web Crypto became stable through global `crypto` in `1.0.0-rc1`:

```javascript
export default function () {
  console.log(crypto.randomUUID());
}
```

Do not import deprecated `k6/experimental/webcrypto`. The crypto module gained
PBKDF2 password-based key derivation in `1.6.0`.

The jslib TOTP package added RFC 6238 generation and verification from a
base32 secret in `1.6.0`:

```javascript
import { TOTP } from 'https://jslib.k6.io/totp/1.0.0/index.js';

const totp = new TOTP('GEZDGNBVGY3TQOJQGEZDGNBVGY3TQOJQ', 6);
const code = await totp.gen();
const valid = await totp.verify(code);
```

## Secret sources and redaction

The asynchronous `k6/secrets` API was introduced in `1.0.0-rc1`. Built-in
sources read key-value files or CLI values, and extensions can supply other
sources. Retrieved values are redacted from logs, including when nested in
logged responses:

```javascript
import secrets from 'k6/secrets';

export default async function () {
  console.log(await secrets.get('cool'));
}
```

```sh
k6 run --secret-source=mock=cool="not cool secret" script.js
```

Redaction expanded to `float32` and `float64` values in `1.0.0-rc2`.

URL-based sources were added in `1.5.0`, but that release supplied only a mock
implementation and no production-ready external secret-manager integration.

`K6_SECRET_SOURCE` became an equivalent to `--secret-source` in `1.7.0` and
accepts the same syntax:

```sh
K6_SECRET_SOURCE='mock=cool="not cool secret"' k6 run script.js
```

In `2.0.0`, `k6 cloud run --local-execution` automatically enables the Cloud
secret source. Use `--no-cloud-secrets` to opt out.

## Assertions

The URL-hosted `k6-testing` preview library became usable in `1.2.0`, bringing
`expect()` and Playwright-style matchers to protocol and browser tests:

```javascript
import { expect } from 'https://jslib.k6.io/k6-testing/0.5.0/index.js';
import http from 'k6/http';

export default function () {
  expect(http.get('https://quickpizza.grafana.com/').status).toBe(200);
}
```

It was a preview with incomplete matcher and feature coverage; pin its URL
version and validate required matchers before relying on it.

## Explicit test failure

`execution.test.fail()` was added in `1.0.0-rc2`. It marks the test failed
without stopping execution, so metrics and cleanup continue:

```javascript
import exec from 'k6/execution';

export default function () {
  exec.test.fail('validation failed');
}
```

Execution-status consumers can distinguish this outcome using
`ExecutionStatusMarkedAsFailed` as of `1.6.0`.

## gRPC values and headers

As of `1.2.0`, gRPC marshals `NaN` and `Infinity` float values as their string
representations rather than `null`; existing scripts need no change.

The gRPC module also supports the `authority` pseudo-header as of `1.2.0` for
services that require it.

## WebSockets

The experimental WebSockets API gained close codes and reasons in `1.5.0`.
`close(code, reason)` sends both values, and the close event exposes them:

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

The module stabilized unchanged at `k6/websockets` in `1.6.0`; migrate off
the deprecated experimental import.

In `1.8.0`, sending a TypedArray through `k6/websockets` began incrementing
`bufferedAmount` correctly, preventing the counter from becoming negative
during transmission.

## Console inspection

`console.log()` began deeply traversing nested arrays and objects in `1.5.0`.
Functions and classes render as `"[object Function]"`, and circular references
as `"[Circular]"`, instead of collapsing the full value to `[object Object]`.

In `1.6.0`, logging also began rendering `ArrayBuffer` bytes and typed-array
types, lengths, and values, including nested binary values.

## HTTP method signatures

Beginning in `1.8.0`, `http.get()` and `http.head()` warn about extra
positional arguments. The extras are still ignored, but callers should move
supported options into the documented parameter object instead of suppressing
the warning.
