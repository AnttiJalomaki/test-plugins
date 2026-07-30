# Caching, environment, and package managers

Use this reference when diagnosing cache misses or incorrect cache hits,
configuring local cache retention, sharing cache across worktrees, or pruning
Bun and Yarn workspaces.

## Environment and cache-setting precedence

`DISPLAY` is passed through by default (since 2.4.0).

Negated `passThroughEnv` entries can exclude both built-in pass-through
variables and variables inherited from `globalPassThroughEnv`. Use negation
when a task must not inherit a repository-wide or built-in value.

Force mode takes precedence over other cache settings (since 2.4.0). In
particular, `remoteCache.enable` no longer overrides force mode. When a forced
run behaves differently from the configured remote-cache policy, treat the
force request as authoritative.

Environment variables used during execution must be declared in task
configuration. Use the Turborepo ESLint rule or Biome's explicitly enabled
`noUndeclaredEnvVars` rule to catch reads that could otherwise produce an
incorrect cache hit.

## Cache sharing across Git worktrees

Linked Git worktrees share their local Turborepo cache automatically (since
2.8.0). No configuration is required:

```bash
turbo run build
git worktree add -B my-branch ../my-branch
cd ../my-branch
turbo run build
```

A result cached in the original worktree can therefore hit in the linked
worktree. Account for this when trying to reproduce a cold-cache run.

## Automatic local cache eviction

Top-level `cacheMaxAge` and `cacheMaxSize` opt into cache eviction (since
2.10.0):

```jsonc
{
  "cacheMaxAge": "7d",
  "cacheMaxSize": "10GB"
}
```

At the start of each `turbo run`, eviction:

1. removes artifacts older than `cacheMaxAge`;
2. removes the oldest remaining artifacts until `cacheMaxSize` is met.

The work runs in a background thread. Omitting the settings leaves their
respective age or size policy disabled.

## Git-aware pruning

Ask `turbo prune` to honor `.gitignore` with `--use-gitignore` (since 2.4.0):

```bash
turbo prune web --use-gitignore
```

Use the target as a positional argument. The older `--scope` form is
deprecated.

The `pruneIncludesGlobalFiles` future flag copies files matched by
`globalDependencies` into prune output (since 2.9.0). Enabling it changes the
global hash.

## Bun pruning and invalidation

`turbo prune` supports Bun 1.2 or newer and its text lockfile (since 2.5.0):

```bash
turbo prune web
```

Stable Bun support parses the text `bun.lock` v1 format granularly (since
2.6.0). A dependency edit invalidates only packages affected by that change
instead of every package in the repository.

Check both the Bun version and lockfile format when pruning or invalidation is
unexpected.

## Yarn catalogs

The lockfile parser understands catalogs from Yarn 4.10.0 and newer (since
Turborepo 2.7.0):

```yaml
catalog:
  react: ^19.2.3
```

Changing a catalog invalidates only affected packages and tasks rather than
the entire repository.

The migration codemod also handles package-manager catalogs (since 2.10.0):

```bash
npx @turbo/codemod migrate
```
