# Execution and Caching

## Watch Mode cache writes

Pass the watched task and explicitly opt into cache writes (since 2.4.0):

```bash
turbo watch dev --experimental-write-cache
```

Do not omit the task argument. The feature remains experimental.

## Persistent sidecars

Use `with` for long-running tasks that must start together (since 2.5.0). This
expresses runtime coexistence instead of the completion ordering imposed by
`dependsOn`:

```json
{
  "tasks": {
    "dev": {
      "with": ["api#start"],
      "persistent": true,
      "cache": false
    }
  }
}
```

Dry-run and summary output include `with` relationships (since 2.6.0).

## Dependency-safe continuation

Use `--continue=dependencies-successful` to continue unrelated tasks after a
failure while running a dependent only if every dependency succeeded (since
2.5.0):

```bash
turbo run test --continue=dependencies-successful
```

## Git-aware pruning

Opt into respecting `.gitignore` during pruning with `--use-gitignore` (since
2.4.0):

```bash
turbo prune web --use-gitignore
```

Use the positional target form; `turbo prune --scope web` is deprecated.

## Package-manager-aware pruning and invalidation

Prune repositories that use Bun 1.2 or newer and its text lockfile normally
(since 2.5.0):

```bash
turbo prune web
```

Stable Bun support parses the text `bun.lock` v1 format granularly (since
2.6.0). A dependency change invalidates only affected packages rather than the
entire repository.

Turborepo also parses catalogs from Yarn 4.10.0 and newer (since 2.7.0). A
catalog edit invalidates only affected packages and tasks:

```yaml
catalog:
  react: ^19.2.3
```

## Cache sharing across Git worktrees

Expect linked Git worktrees to share the local cache automatically (since
2.8.0). No configuration is required; work completed and cached in one worktree
can hit in another.

## Cache command migrations

Prefer explicit cache modes (deprecated forms documented in 2.9.0):

| Deprecated interface | Replacement |
| --- | --- |
| `--no-cache` | `--cache=local:r,remote:r` |
| `TURBO_REMOTE_ONLY` or `--remote-only` | `--cache=remote:rw` |
| `TURBO_REMOTE_CACHE_READ_ONLY` or `--remote-cache-read-only` | `--cache=local:rw,remote:r` |

Remember that force mode overrides other cache settings, including
`remoteCache.enable`.

## Daemon-free execution

Do not rely on a daemon for `turbo run` or `turbo watch` (since 2.9.0). The
following no longer have a role and are deprecated:

- `TURBO_DAEMON`
- `--daemon`
- `--no-daemon`
- the `daemon` configuration key

## Graceful shutdown

Allow task cleanup handlers to finish after `SIGINT` or `SIGTERM` (since
2.10.0). Turborepo forwards the signal and waits. Press `Ctrl+C` a second time
to force an immediate exit; no configuration is needed.

## Automatic local cache eviction

Opt into age- and size-based eviction with top-level settings (since 2.10.0):

```jsonc
{
  "cacheMaxAge": "7d",
  "cacheMaxSize": "10GB"
}
```

At the start of each `turbo run`, a background thread first removes expired
artifacts, then removes the oldest artifacts necessary to meet the size cap.
