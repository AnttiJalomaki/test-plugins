# Static Assets and Pages Migration

Use this reference when publishing static files with a Worker or converting a
Pages project to Workers Static Assets.

## Deployment, matching, and binding

`assets.directory` publishes a directory and Worker code as one deployment.
By default, an exact asset match is served without invoking the Worker; a
miss invokes `main`. Set `assets.binding` when Worker code needs to delegate
to the asset service with `env.ASSETS.fetch(request)`.

```jsonc
{
  "main": "src/index.js",
  "assets": {
    "directory": "./dist",
    "binding": "ASSETS"
  }
}
```

An assets-only Worker must omit `binding`, which is valid only with `main`.
Projects using the Workers Vite plugin need not specify `assets.directory`.

## Fallback and Worker-first routing

Workers does not infer fallback behavior from `index.html` or `404.html` as
Pages does. Set `assets.not_found_handling` to
`single-page-application` or `404-page`.

`assets.run_worker_first` accepts `true` or an ordered list of route patterns.
A leading `!` excludes a matching path.

```jsonc
{
  "assets": {
    "directory": "./dist",
    "not_found_handling": "single-page-application",
    "run_worker_first": ["/api/*", "!/api/docs/*"]
  }
}
```

From compatibility date `2025-04-01`, navigation requests prefer Static
Assets fallback handling even without an exact asset match. SPA `/index.html`
and custom `/404.html` responses therefore run before the Worker. This does
not apply when `assets.run_worker_first = true`.

Worker-first middleware is billed as a normal Worker invocation.

## Wrangler configuration conversion

Replace `pages_build_output_dir` with `assets.directory` in a root Wrangler
configuration. Preserve a Pages Functions project's compatibility date.
Placement and compatibility flags can also be carried into the Worker config.

```jsonc
{
  "name": "my-worker",
  "compatibility_date": "2026-07-28",
  "assets": { "directory": "./dist/client/" }
}
```

Workers does not automatically exclude `node_modules`, `.DS_Store`, `.git`,
or similar files as Pages does. Put `.assetsignore` inside the configured
asset directory:

```text
**/node_modules
**/.DS_Store
**/.git
_worker.js
```

## Pages Functions conversion

For advanced-mode Pages, move `_worker.js` outside the asset directory or
exclude it through `.assetsignore`, then point `main` to it.

A Pages `functions/` directory must first be compiled into one entrypoint:

```sh
wrangler pages functions build --outdir=./dist/worker/
```

Set `main` to `./dist/worker/index.js`.

Pages `_routes.json` and middleware do not automatically preserve
function-first behavior. Configure `assets.run_worker_first` for
authentication, logging, or other middleware that must run before assets.

## Development and builds

Replace `wrangler pages dev` and `wrangler pages deploy` with `wrangler dev`
and `wrangler deploy`. The default development ports differ: Pages uses 8788,
while Workers uses 8787.

For Workers Builds, connect the repository and disable Pages automatic
deployments. Build-time variables are configured separately from Worker
runtime variables.

## Preview environments

Enable `preview_urls` and non-production branch builds in Workers Builds to
approximate Pages branch previews:

```jsonc
{
  "preview_urls": true
}
```

Workers does not natively provide separate production and non-production
bindings. Use Wrangler environments and matching build configuration when
that separation is required.

Workers Builds has less-configurable non-production branch controls than
Pages, and custom branch aliases are not supported.

## Headers, redirects, domains, and Early Hints

Workers Static Assets honors Pages-style `_headers` and `_redirects` when the
files remain inside the asset directory.

Set `workers_dev: true` to opt into the account's `workers.dev` subdomain.
Worker custom domains require Cloudflare-managed nameservers. Use a Worker
route when only selected paths should migrate.

Workers Early Hints requires the zone setting and appropriate `Link` headers.
