# Extensions and dependencies

## Automatic resolution

In 1.0.0-rc2, experimental Cloud binary provisioning could be enabled with
`K6_BINARY_PROVISIONING=true`. It requested a custom binary containing
supported extensions for `k6 cloud`; it did not apply to `k6 run`. Local
execution still required a Cloud token from the normal login flow.

In 1.2.0, renamed automatic extension resolution became enabled by default.
k6 detects extension imports and provisions a matching binary. At that point,
community-list extensions required `K6_ENABLE_COMMUNITY_EXTENSIONS=true` and
worked only with local `k6 run` or `k6 cloud run --local-execution`; Cloud
execution accepted only official extensions.

In 2.0.0, both `K6_BINARY_PROVISIONING` and
`K6_ENABLE_COMMUNITY_EXTENSIONS` were removed. Community extensions resolve
through the default build service. `K6_AUTO_EXTENSION_RESOLUTION` is needed
only to disable resolution explicitly.

Since 1.8.0, normal k6 logs expose artifact resolution, cache hits, downloads,
retries, and cache pruning at their corresponding levels.

## Import discovery

Since 1.4.0, automatic resolution discovers extensions only from ES module
`import`, not CommonJS `require()`. CommonJS code can use a directive at the
beginning of every relevant file, preceded only by an optional shebang,
whitespace, or comments:

```javascript
"use k6 with k6/x/redis"
const redis = require('k6/x/redis');
```

## Dependency inspection and archives

Since 1.6.0, `k6 deps` reports a script or archive's dependencies; use
`--json` for automation:

```sh
k6 deps --json script.js
```

It detects static imports, not dynamic `require()`. Use
`K6_DEPENDENCIES_MANIFEST` to constrain a detected dependency that has no
version pragma:

```sh
K6_DEPENDENCIES_MANIFEST='{"k6/x/faker":">=v0.4.4"}' k6 run script.js
```

Since 2.0.0, `k6 archive` records constraints for pre-manifest `k6/x/`
dependencies in the `dependencies` field of `metadata.json`, preserving them
for automatic resolution when the archive is run again.

## Extension CLI commands

Since 1.5.0, extensions can register commands in the `k6 x` namespace:

```sh
k6 x my-tool --help
```

Since 1.7.0, k6 provisions a suitable binary automatically when a requested
`k6 x` command needs an extension absent from the current binary.

Since 2.1.0, running `k6 x` lists commands compiled into the binary and those
advertised by the official and community registry. After one invocation
caches the catalog, tab completion exposes the same commands.

Provisioned subcommands receive the host version through
`K6_PROVISION_HOST_VERSION` since 2.0.0, allowing their documentation and
behavior to match the invoking k6.

## Extension runtime integration

Since 1.5.0, extensions can use k6's DNS resolver and therefore honor `hosts`
overrides, custom DNS servers, and cache settings.

The official `k6/x/dns` extension became available through automatic
resolution in 1.4.0 without a custom build. It supports A and AAAA lookups:

```javascript
import dns from 'k6/x/dns';

export default function () {
  console.log(dns.resolve('grafana.com', { recordType: 'A' }));
}
```

`k6/experimental/redis` was deprecated in 1.5.0; migrate to the official Redis
extension.

## Go extension migration

In 2.0.0:

- Change every k6 Go import to the `go.k6.io/k6/v2` module path.
- Public k6 Go types no longer expose easyjson-generated `MarshalJSON` and
  `UnmarshalJSON`; use standard `encoding/json`.
