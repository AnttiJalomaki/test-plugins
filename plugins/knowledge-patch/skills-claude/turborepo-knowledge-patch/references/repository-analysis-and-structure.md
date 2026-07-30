# Repository analysis and structure

Use this reference when checking package boundaries, querying repository
state, narrowing affected work, serving local microfrontends, or operating
non-JavaScript workspace layouts.

## Package boundaries

Run the experimental boundary checker to detect imports that escape a
package's directory and imports of packages missing from that package's
dependencies (since 2.4.0):

```bash
turbo boundaries
```

Package rules, implicit-dependency handling, and TypeScript configuration path
aliases are supported (since 2.5.0). Boundary analysis also detects circular
package dependencies and follows dynamic imports (since 2.10.0).

Boundary findings concern package structure. For execution failures in a
cyclic package graph, inspect the Task Graph separately; package cycles are
allowed when they do not create cyclic task relationships.

## Stable repository queries

`turbo query` is stable (since 2.9.0). Calling it without a query opens
GraphiQL. Supply GraphQL inline or through `--file`, and inspect the available
GraphQL schema with `--schema`:

```bash
turbo query
turbo query --schema
turbo query '{ packages { items { name } } }'
turbo query --file=query.gql
```

The `affected` shorthand returns structured JSON for changed tasks or
packages:

```bash
turbo query affected --tasks build
turbo query affected --packages
```

Use this instead of the deprecated `turbo-ignore`.

`turbo query ls` pretty-prints package details by default and supports JSON,
affected-only output, selectors, and filters:

```bash
turbo query ls web --output=json
turbo query ls --affected --filter='./apps/*'
```

The JSON output from `turbo ls` includes dependents (since 2.6.0).

## Intersect affected and filtered scopes

`--affected` and `--filter` may be combined (since 2.10.0). Selection is the
intersection, not the union:

```bash
turbo run build --affected --filter=web
turbo run build --affected --filter=!docs
turbo query ls --affected --filter=my-app
```

The negated filter removes packages from the affected set. Future flags can
change affected and filter resolution from package-level to task-level
semantics, so record those flags when comparing selection results.

## Local microfrontend proxy

Turborepo can serve several applications behind one local proxy at
`localhost:3024` (since 2.6.0). Put `microfrontends.json` in the parent
application and map development ports and route prefixes:

```json
{
  "$schema": "https://turborepo.dev/microfrontends/schema.json",
  "applications": {
    "web": {
      "development": {
        "local": 3000
      }
    },
    "docs": {
      "development": {
        "local": 3001
      },
      "routing": [
        {
          "paths": ["/docs", "/docs/:path*"]
        }
      ]
    }
  }
}
```

Start it with:

```bash
turbo dev
```

The unrouted application handles all paths not claimed by a routing entry.

## Graph inspection

`turbo devtools` shows live Package Graph and Task Graph views that hot-reload
as repository files change (since 2.7.0). Direct and transitive relationships
help explain cache misses:

```bash
turbo devtools
```

For saved graph output, prefer SVG, HTML, Mermaid, or DOT. PNG, JPG, and PDF
output is deprecated; JSON graph output has moved to `turbo query` (since
2.9.0).

## Catalogs and Cargo-only repositories

The migration codemod handles package-manager catalogs (since 2.10.0):

```bash
npx @turbo/codemod migrate
```

Turborepo also supports a repository that contains only a Cargo workspace
(since 2.10.0). It can infer tasks for Cargo workspace members, so a JavaScript
workspace manifest is not required merely to establish repository structure.
