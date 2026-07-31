# Extensions and Dependencies

## Prefer stable built-in modules

`k6/browser`, `k6/net/grpc`, and `k6/crypto` are stable and production-ready
(1.0.0). Use their stable paths instead of treating them as experimental
interfaces.

WebSockets became stable at `k6/websockets` in 1.6.0. The API did not change,
but `k6/experimental/websockets` is deprecated and will be removed:

```javascript
import ws from 'k6/websockets';
```

`k6/experimental/redis` is deprecated (1.5.0). Migrate Redis tests to the
official k6 Redis extension rather than adding new use of the experimental
module.

## Let k6 resolve extensions

Automatic extension resolution detects extension imports and provisions a
matching binary by default (since 1.2.0). In that release, extensions from the
community list required opt-in and were supported only for local `k6 run` or
`k6 cloud run --local-execution`; Cloud execution permitted only official
extensions:

```sh
K6_ENABLE_COMMUNITY_EXTENSIONS=true k6 run script.js
```

For k6 v2, remove `K6_BINARY_PROVISIONING` and
`K6_ENABLE_COMMUNITY_EXTENSIONS` (2.0.0). Community extensions resolve through
the default build service. `K6_AUTO_EXTENSION_RESOLUTION` is needed only when
explicitly disabling resolution.

Provisioning emits ordinary k6 logs for resolution, cache hits, downloads,
retries, and pruning (1.8.0). Use those log levels to diagnose failures.

## Make dependencies discoverable

### Use static ES module imports

Automatic extension discovery follows ES module `import` statements, not
CommonJS `require()` calls (since 1.4.0). For CommonJS, a directive can declare
the extension, but it must be at the start of each relevant file after only an
optional shebang, whitespace, or comments:

```javascript
"use k6 with k6/x/redis"
const redis = require('k6/x/redis');
```

### Inspect and constrain dependencies

`k6 deps` reports dependencies for a script or archive, and `--json` supports
automation (since 1.6.0):

```sh
k6 deps --json script.js
K6_DEPENDENCIES_MANIFEST='{"k6/x/faker":">=v0.4.4"}' k6 run script.js
```

Like automatic resolution, dependency inspection finds static imports but not
dynamic `require()` calls. Use `K6_DEPENDENCIES_MANIFEST` to provide constraints
for discovered dependencies that have no version pragma.

### Preserve dependencies in archives

In v2, `k6 archive` records pre-manifest `k6/x/` constraints in the
`dependencies` field of `metadata.json` (2.0.0). This lets automatic resolution
recover extension imports when the archive is run later. Inspect this field
when a source run succeeds but an archived run provisions a different binary.

## Use official extensions

The officially supported `k6/x/dns` extension can be imported through
automatic resolution without a custom build (since 1.4.0). It supports A and
AAAA lookups:

```javascript
import dns from 'k6/x/dns';

export default function () {
  const answer = dns.resolve('grafana.com', { recordType: 'A' });
  console.log(answer.records.map(({ address }) => address).join(', '));
}
```

Extensions can use k6's own DNS resolver (since 1.5.0). Resolution then honors
the test's `hosts` overrides, custom DNS servers, and DNS cache settings; an
extension does not need to reimplement those settings.

## Build extension subcommands

Extensions can register commands beneath `k6 x` (since 1.5.0):

```sh
k6 x my-tool --help
```

When a `k6 x` command requires an extension missing from the current binary,
k6 provisions and runs a suitable binary automatically (since 1.7.0); a manual
`xk6` build is unnecessary:

```sh
k6 x httpbin
```

Provisioned subcommands receive the invoking k6 version through
`K6_PROVISION_HOST_VERSION` in v2 (2.0.0). Use it when an extension command must
select compatible documentation or behavior.

Running `k6 x` lists subcommands compiled into the binary and those advertised
by the official and community registry (since 2.1.0). Tab completion exposes
the same commands after a prior invocation has cached the catalog locally.

## Migrate Go extensions to v2

k6 v2 uses the Go module path `go.k6.io/k6/v2` (2.0.0). Update every k6 import:

```go
import "go.k6.io/k6/v2/js/modules"
```

Public k6 Go types no longer supply easyjson-generated `MarshalJSON` and
`UnmarshalJSON` methods in v2. Extensions depending on those methods must use
standard `encoding/json` marshaling (2.0.0).
