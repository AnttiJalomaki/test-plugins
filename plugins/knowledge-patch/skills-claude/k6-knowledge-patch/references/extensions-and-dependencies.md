# Extensions and Dependencies

## Automatic extension resolution

The feature first exposed as experimental Cloud binary provisioning in
`1.0.0-rc2` became automatic extension resolution in `1.2.0`. k6 detects
extension imports and provisions a matching binary, removing the ordinary need
for a manual xk6 build.

In `1.2.0`, extensions from the community list required
`K6_ENABLE_COMMUNITY_EXTENSIONS=true` and worked only with local `k6 run` or
`k6 cloud run --local-execution`. Cloud execution accepted official
extensions only:

```sh
K6_ENABLE_COMMUNITY_EXTENSIONS=true k6 run script.js
```

Those opt-in rules are historical. In `2.0.0`,
`K6_BINARY_PROVISIONING` and `K6_ENABLE_COMMUNITY_EXTENSIONS` were removed,
and community extensions began resolving through the default build service.
Set `K6_AUTO_EXTENSION_RESOLUTION` only to disable automatic resolution.

Provisioning emits ordinary k6 log entries as of `1.8.0`, including artifact
resolution, cache hits, downloads, retries, and cache pruning at the
corresponding log levels. Use these logs to diagnose resolution failures.

## Make dependencies discoverable

Automatic resolution stopped following CommonJS `require()` calls in `1.4.0`.
Use static ES module imports:

```javascript
import redis from 'k6/x/redis';
```

CommonJS files can declare an otherwise invisible dependency with a directive:

```javascript
"use k6 with k6/x/redis"
const redis = require('k6/x/redis');
```

The directive must appear at the beginning of each relevant file, preceded
only by an optional shebang, whitespace, or comments.

## Inspect and constrain dependencies

`k6 deps` was added in `1.6.0` for scripts and archives:

```sh
k6 deps script.js
k6 deps --json script.js
```

Like automatic resolution, it recognizes static imports but not dynamic
`require()` calls. Supply version constraints for detected dependencies that
lack a version pragma with `K6_DEPENDENCIES_MANIFEST`:

```sh
K6_DEPENDENCIES_MANIFEST='{"k6/x/faker":">=v0.4.4"}' k6 run script.js
```

Beginning in `2.0.0`, `k6 archive` writes constraints for pre-manifest
`k6/x/` imports to the `dependencies` field of `metadata.json`. This retains
the information needed to resolve extensions when the archive runs later.

## Extension subcommands

Extensions gained a common CLI namespace in `1.5.0`:

```sh
k6 x my-tool --help
```

As of `1.7.0`, k6 automatically provisions a suitable binary when a requested
`k6 x` command is absent from the current binary:

```sh
k6 x httpbin
```

The invoked subcommand receives the host k6 version in
`K6_PROVISION_HOST_VERSION` as of `2.0.0`, allowing it to choose compatible
behavior or documentation.

Running `k6 x` in `2.1.0` lists both compiled-in subcommands and commands
advertised by the official and community extension registry. A prior
invocation caches the catalog used for tab completion.

## Official DNS and Redis extensions

The official `k6/x/dns` extension became available through automatic
resolution in `1.4.0` and supports A and AAAA lookups:

```javascript
import dns from 'k6/x/dns';

export default function () {
  const answer = dns.resolve('grafana.com', { recordType: 'A' });
  console.log(answer.records.map(({ address }) => address).join(', '));
}
```

`k6/experimental/redis` was deprecated in `1.5.0`; use the official Redis
extension.

## DNS behavior inside extensions

Extensions can use k6's own DNS resolver as of `1.5.0`. Doing so makes
extension lookups honor test `hosts` overrides, custom DNS servers, and DNS
cache settings instead of implementing separate resolution.

## Updating Go extensions for v2

In `2.0.0`, update all Go imports to `go.k6.io/k6/v2/...`. Public k6 Go types
also no longer expose easyjson-generated marshal methods; use
`encoding/json`. See
[migrations-and-compatibility.md](migrations-and-compatibility.md) for the
complete migration checklist.

