# Static Assets and Pages migration

## Deploy assets with Worker code

`assets.directory` publishes a directory and Worker code as one deployment
(batch `static-assets-migration`).

By default, an exact asset match is served without invoking the Worker. A miss
invokes `main`. Adding `assets.binding` lets Worker code delegate to the asset
service:

```jsonc
{
  "main": "src/index.js",
  "assets": {
    "directory": "./dist",
    "binding": "ASSETS"
  }
}
```

```js
return env.ASSETS.fetch(request);
```

An assets-only Worker must omit `binding`, which is valid only when `main` is
present. Projects using the Cloudflare Vite plugin do not need to set
`assets.directory`.

## Configure fallback and Worker-first routing

Workers does not infer fallback mode from the presence of `index.html` or
`404.html` as Pages does. Set `assets.not_found_handling` to
`single-page-application` or `404-page`.

`assets.run_worker_first` accepts `true` or an ordered list of route patterns.
A leading `!` excludes a matching path:

```jsonc
{
  "assets": {
    "directory": "./dist",
    "not_found_handling": "single-page-application",
    "run_worker_first": ["/api/*", "!/api/docs/*"]
  }
}
```

From compatibility date `2025-04-01`, navigation requests prefer Static Assets
fallback even without an exact match. SPA `/index.html` and custom `/404.html`
responses therefore run before the Worker unless
`assets.run_worker_first = true` or a matching Worker-first rule applies.

Worker-first middleware is billed as a normal Worker invocation.

## Convert the Pages configuration

Replace `pages_build_output_dir` with `assets.directory` in a root Wrangler
configuration. Preserve the Pages Functions project's compatibility date.
Placement and compatibility flags can carry over:

```jsonc
{
  "name": "my-worker",
  "compatibility_date": "2026-07-28",
  "assets": {
    "directory": "./dist/client/"
  }
}
```

## Exclude files explicitly

Workers does not automatically apply Pages exclusions for `node_modules`,
`.DS_Store`, `.git`, or generated Worker files. Put `.assetsignore` inside the
configured asset directory:

```text
**/node_modules
**/.DS_Store
**/.git
_worker.js
```

## Convert Pages Functions

For an advanced-mode project, move `_worker.js` outside the asset directory or
exclude it with `.assetsignore`, then point `main` to it.

A `functions/` directory must first be compiled to one Worker entrypoint:

```sh
wrangler pages functions build --outdir=./dist/worker/
```

Set `main` to `./dist/worker/index.js`.

Pages `_routes.json` and middleware do not automatically preserve their
function-first behavior. Use `assets.run_worker_first` for authentication,
logging, and other middleware that must run before assets.

## Change development and deployment commands

Replace:

- `wrangler pages dev` with `wrangler dev`;
- `wrangler pages deploy` with `wrangler deploy`.

The default local port changes from Pages port 8788 to Workers port 8787.

For Workers Builds, connect the repository and disable Pages automatic
deployments. Build-time variables in Workers Builds are configured separately
from Worker runtime variables.

## Recreate preview environments

To approximate Pages branch previews, enable both `preview_urls` and
non-production branch builds in Workers Builds:

```jsonc
{
  "preview_urls": true
}
```

Workers does not natively provide separate production and non-production
bindings. Use Wrangler environments and the corresponding build configuration
when the bindings must differ.

Non-production branch controls are less configurable than Pages, and custom
branch aliases are not supported in the source version.

## Preserve headers, redirects, and routing

Workers Static Assets supports Pages-style `_headers` and `_redirects` files
when they remain inside the asset directory.

Set `workers_dev: true` to opt into the account's `workers.dev` subdomain.
Worker custom domains require Cloudflare-managed nameservers. Use a Worker route
when only selected paths should migrate.

Workers Early Hints require both the zone setting and appropriate `Link`
headers.
