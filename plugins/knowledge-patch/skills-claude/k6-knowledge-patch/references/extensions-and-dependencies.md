# Extensions and Dependencies

## Automatic extension resolution

### Provision imports automatically (since 1.2.0)

Automatic extension resolution, previously called Binary Provisioning, became
enabled by default. k6 detects extension imports and provisions a matching
binary instead of requiring a manual custom build.

In 1.2, community-list extensions required an opt-in and worked only with
local `k6 run` or `k6 cloud run --local-execution`; Cloud execution allowed
only official extensions:

```sh
K6_ENABLE_COMMUNITY_EXTENSIONS=true k6 run script.js
```

For k6 v2, do not retain that environment variable: community extensions
resolve through the default build service.

### Use static imports or an explicit directive (since 1.4.0)

Resolution discovers ES module `import` statements, not CommonJS `require()`
calls. CommonJS files can declare an extension with a `"use k6 with ..."`
directive. Put it at the beginning of every relevant file, preceded only by an
optional shebang, whitespace, or comments.

```javascript
"use k6 with k6/x/redis"
const redis = require('k6/x/redis');
```

### Remove obsolete provisioning switches in v2 (since 2.0.0)

`K6_BINARY_PROVISIONING` and `K6_ENABLE_COMMUNITY_EXTENSIONS` were removed.
Automatic resolution uses the default build service. Set
`K6_AUTO_EXTENSION_RESOLUTION` only when resolution must be disabled
explicitly.

### Diagnose provisioning from normal logs (since 1.8.0)

Automatic provisioning emits k6 log entries for artifact resolution, cache
hits, downloads, retries, and cache pruning. Each entry uses the corresponding
log level, so ordinary k6 logging can diagnose the provisioning path.

## Dependency inspection and archives

### Inspect static dependencies (since 1.6.0)

`k6 deps` reports dependencies for a script or archive and supports `--json`
for automation. Like automatic resolution, it detects static imports but not
dynamic `require()` calls.

Use `K6_DEPENDENCIES_MANIFEST` to constrain detected dependencies that do not
have a version pragma.

```sh
k6 deps --json script.js
K6_DEPENDENCIES_MANIFEST='{"k6/x/faker":">=v0.4.4"}' k6 run script.js
```

### Preserve extension dependencies in archives (since 2.0.0)

`k6 archive` records pre-manifest `k6/x/` dependency constraints in the
`dependencies` field of `metadata.json`. When the archive runs later, that
metadata preserves its extension imports for automatic resolution.

## Extension subcommands

### Register commands under `k6 x` (since 1.5.0)

Extensions can register CLI utilities under the consistent `k6 x` namespace:

```sh
k6 x my-tool --help
```

### Provision missing subcommands (since 1.7.0)

When a `k6 x` command needs an extension absent from the current binary, k6
provisions a suitable binary and runs the command. A manual `xk6` build is not
required.

```sh
k6 x httpbin
```

### Receive the invoking host version (since 2.0.0)

Provisioned subcommands receive the invoking k6 version in
`K6_PROVISION_HOST_VERSION`. Extension commands can use it to select compatible
documentation or behavior.

### Discover registry and compiled commands (since 2.1.0)

Run `k6 x` without a subcommand to list commands compiled into the binary and
commands advertised by the official and community extension registry. After
one invocation caches the catalog locally, tab completion exposes the same
commands.

## Extension DNS behavior

### Use k6's configured resolver (since 1.5.0)

Extensions can use k6's resolver so their lookups honor the test's `hosts`
overrides, custom DNS servers, and DNS cache settings. Extension code need not
reimplement those settings.

### Use the official DNS extension (since 1.4.0)

The supported `k6/x/dns` extension resolves A and AAAA records and works with
automatic resolution without a custom k6 build.

```javascript
import dns from 'k6/x/dns';

export default function () {
  const answer = dns.resolve('grafana.com', { recordType: 'A' });
  console.log(answer.records.map(({ address }) => address).join(', '));
}
```

## Redis migration

### Leave the experimental Redis module (since 1.5.0)

`k6/experimental/redis` is deprecated and scheduled for removal. Migrate Redis
tests to the official k6 Redis extension.

## Go extension compatibility

### Meet the compiler floor

The build minimum rose to Go 1.24 in k6 1.4.0. Since 1.7.0, building k6
requires Go 1.25 or newer and the default Go toolchain is 1.26.

### Update the Go module path for v2 (since 2.0.0)

k6 v2 uses `go.k6.io/k6/v2`. Update every import in extensions and external Go
packages:

```go
import "go.k6.io/k6/v2/js/modules"
```

### Use standard JSON marshaling in v2 (since 2.0.0)

Public k6 Go types no longer provide easyjson-generated `MarshalJSON` and
`UnmarshalJSON` methods. Extensions that relied on them must marshal and
unmarshal with the standard `encoding/json` package.
