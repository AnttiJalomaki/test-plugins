# Ecosystem, Documentation, and Migrations

## Local microfrontend proxy

Serve multiple local applications through one proxy at `localhost:3024` (since
2.6.0). Place `microfrontends.json` in the parent application, map each
application to its development port and optional route prefixes, and run
`turbo dev`. The application without matching routes handles the remaining
paths.

```json
{
  "$schema": "https://turborepo.dev/microfrontends/schema.json",
  "applications": {
    "web": {
      "development": { "local": 3000 }
    },
    "docs": {
      "development": { "local": 3001 },
      "routing": [
        { "paths": ["/docs", "/docs/:path*"] }
      ]
    }
  }
}
```

```bash
turbo dev
```

## Agent guidance

Install the official Turborepo Agent Skill when additional monorepo patterns and
anti-patterns would help (since 2.8.0):

```bash
npx skills add vercel/turborepo
```

## Machine-readable documentation

Request documentation as Markdown with `Accept: text/markdown`, append `.md` to
a documentation route, or discover pages from `/sitemap.md` (since 2.8.0).
Use a version subdomain such as `v2-7-6.turborepo.dev` when documentation must
match a pinned release.

```bash
curl -sL -H "Accept: text/markdown" https://turborepo.dev/repo/docs
curl -sL https://turborepo.dev/sitemap.md
```

Search documentation from the terminal with `turbo docs`:

```bash
turbo docs "package configurations"
```

## Migration deprecations

Replace warning-producing interfaces documented in 2.9.0 before the next major
upgrade:

- Treat `turbo scan` as obsolete; it has no replacement.
- Replace `--parallel` with task-level `persistent` and `with`.
- Replace `--no-cache` with `--cache=local:r,remote:r`.
- Replace `TURBO_REMOTE_ONLY` and `--remote-only` with
  `--cache=remote:rw`.
- Replace `TURBO_REMOTE_CACHE_READ_ONLY` and `--remote-cache-read-only` with
  `--cache=local:rw,remote:r`.
- Replace `.png`, `.jpg`, or `.pdf` graph output with `.svg`, `.html`,
  `.mermaid`, or `.dot`.
- Replace `.json` graph output with `turbo query`.
- Replace `turbo prune --scope web` with `turbo prune web`.

## Catalog-aware upgrades

Run the migration codemod normally when package-manager catalogs are present
(since 2.10.0); the codemod preserves and updates catalog-aware repositories:

```bash
npx @turbo/codemod migrate
```

## Cargo-only workspaces

Use Turborepo in repositories that contain only a Cargo workspace (since
2.10.0). It recognizes the repository and infers tasks for Cargo workspace
members without requiring a JavaScript workspace.
