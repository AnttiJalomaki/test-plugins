# Programmatic APIs

## Import Stable Entry Points

Only documented package subpaths are stable public APIs. Do not import CLI
command internals. Registry schemas are exposed from `shadcn/schema`:

```ts
import { registryItemSchema } from "shadcn/schema"
```

Direct API consumers should replace earlier names:

- `fetchRegistry` becomes `getRegistry`.
- `resolveRegistryTree` becomes `resolveRegistryItems`.

Existing `components.json` files and installed components remain compatible;
the rename matters to code that calls the package API directly.

## Resolve Registry Configuration

`getRegistriesConfig(cwd)` reads `components.json`. If that file is absent, it
falls back to a top-level `registries` entry in `package.json`.

Registry fetches use resolved-URL caching for the lifetime of the process by
default and deduplicate concurrent in-flight requests. Disable caching when a
server or watcher must observe upstream changes immediately:

```ts
const config = await getRegistriesConfig(process.cwd())
const items = await getRegistryItems(["@acme/button"], {
  config,
  useCache: false,
})
```

## Install Without Prompts

`addRegistryItems` applies files, dependencies, environment variables, CSS,
and Tailwind configuration without prompting. It throws errors rather than
exiting the host process. Existing files are skipped unless `overwrite` is
enabled.

The function does not load project configuration. Pass a resolved config that
contains aliases and `resolvedPaths`:

```ts
const cwd = process.cwd()
const config = await getRegistriesConfig(cwd)

await addRegistryItems(["@acme/agent"], {
  cwd,
  config,
  overwrite: false,
  silent: true,
})
```

A registries-only config is sufficient only for universal `registry:item` or
`registry:file` payloads whose files all have explicit targets.

## Handle Typed Failures

Registry functions throw `RegistryError` subclasses. Recover by a specific
error class or `RegistryErrorCode` for failures involving missing items,
authentication, fetches, configuration, local files, parsing, validation,
invalid namespaces, or missing environment variables.

```ts
try {
  await getRegistry("@unknown")
} catch (error) {
  if (error instanceof RegistryNotFoundError) {
    // Recover from an unknown registry.
  }
}
```

Do not assume library calls terminate the process; decide at the application
boundary whether to retry, report, skip, or fail.

## Encode and Decode Presets

Use the `shadcn/preset` entry point for programmatic preset codes:

```ts
import { decodePreset, encodePreset } from "shadcn/preset"

const code = encodePreset({
  style: "vega",
  theme: "blue",
  radius: "large",
})

const preset = decodePreset(code)
```

`encodePreset` accepts a partial preset, fills omitted values from
`DEFAULT_PRESET_CONFIG`, and returns a version-prefixed, URL-safe code.
`decodePreset` returns the fully defaulted configuration, or `null` when the
code is missing or invalid.

The same entry point exports preset validators, random-preset helpers, Base62
helpers, and the `PRESET_*` option constants used by theme tooling.
