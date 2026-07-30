# Wrangler and deployment

## Wrangler v4 migration

Wrangler v4 follows the Node.js release lifecycle and drops Node.js 16 support
(batch `2025`). Upgrade the Node runtime before adopting it.

Wrangler's bundled esbuild changes from 0.17.19 to 0.24. Wildcard dynamic
imports now bundle every matching file. Because esbuild remains pre-1.0,
Wrangler minor releases may update it in ways that alter bundling; inspect and
test production artifacts when updating Wrangler.

### Removed and deprecated interfaces

Use these replacements:

| Old interface | Current interface |
|---|---|
| `legacy_assets` | Static Assets |
| `node_compat` | `nodejs_compat` |
| `getBindingsProxy()` | `getPlatformProxy()` |
| `publish` | `deploy` |
| `pages publish` | `pages deploy` |
| `generate` | `npm create cloudflare@latest` |
| `wrangler version` | `wrangler --version` |

Delete `usage_model`; it was ineffective. Workers Sites is deprecated in favor
of Static Assets. Service environments configured with `legacy_env` are
deprecated in favor of Wrangler environments.

## Local and remote resource commands

Every Wrangler command that can operate locally or remotely defaults to local
mode. A script intended to read or mutate account KV or R2 data must pass
`--remote` explicitly:

```sh
wrangler kv key get --binding MY_KV "my-key" --remote
```

Do not confuse this CLI default with the Vite plugin's remote-binding default.

## Named authentication profiles

Wrangler supports named OAuth logins. Activate a profile for a directory and
its descendants, or select it explicitly:

```sh
wrangler auth create client-a
wrangler auth activate client-a ~/clients/client-a
wrangler deploy --profile client-a
```

An `account_id` remains useful when the project must be constrained to the
intended account. In CI and other automation, `CLOUDFLARE_API_TOKEN` takes
precedence over profiles (batch `2026`).

## Configuration-derived runtime types

`wrangler types` generates `worker-configuration.d.ts` from the Worker
configuration, including:

- compatibility date;
- compatibility flags;
- bindings;
- module rules.

Include the generated declaration through `compilerOptions.types`. Add
`@types/node` when the Worker uses `nodejs_compat`. If the declaration is
committed, check it for drift in CI:

```sh
wrangler types --check
```

`@cloudflare/workers-types` v5 has a different role: its root exports only the
latest stable types, and `/experimental` exports experimental APIs. Dated
package entrypoints have been removed.

## Deployment reminders

- Use `wrangler deploy`, not the removed `publish` command.
- Use `wrangler pages deploy` only while operating a Pages project; a migrated
  Workers Static Assets project uses `wrangler deploy`.
- When using the Vite plugin, `wrangler deploy` deploys only the entry Worker.
  Deploy every auxiliary Worker from its generated configuration separately.
- Authentication selection and remote-resource selection are independent:
  choosing `--profile` does not imply `--remote`.
