# Architecture and migration

Use this reference to choose the correct framework line, understand Remix 3's
design constraints, scaffold a beta project, and migrate imports and generated
application structure.

## Distinguish the React and Remix 3 paths

Remix v1/v2 continues through React Router v7 framework mode. That line
absorbed the Remix bundler and server runtime and is the stable,
long-term-supported choice for React applications.

Remix 3 is a separate framework rebuilt around its own web-oriented component
model. It is not the once-planned RSC-first React successor and does not retain
React as a critical dependency. At the initial direction announcement, no
preview release existed. Source batch: `remix-3-direction`.

React Router v7 offers the incremental React path. Its then-previewed RSC
support allows loaders and actions to return server components and supports
server-only routes. This lets a React application adopt server-component
architecture gradually without waiting for Remix 3.

## Remix 3 design constraints

The planned toolkit emphasizes:

- Web APIs and JavaScript as the portability contract;
- standalone packages that compose without forcing the whole framework;
- runtime execution without a mandatory bundler, compiler, type generator, or
  other static-analysis stage;
- `--import` loaders as the exception for TypeScript and JSX transforms;
- no critical dependencies and a zero-dependency goal; and
- independently useful packages re-exported together from a single `remix`
  package in the original direction.

The later beta packaging supersedes that early re-export intention: the root
`remix` export is gone, and code imports domain-oriented subpaths instead. This
is an API evolution, not a reason to preserve root imports.

## Beta suitability and scaffolding

The first cohesive Remix 3 beta preview supports experiments and prototypes but
is explicitly not production-ready. It brings routing, sessions,
authentication, forms, uploads, static and compiled assets, data access, server
rendering, and UI into one preview. Source batch: `remix-3-beta-preview`.

Scaffold from the `next` release line:

```sh
npx remix@next new my-remix-app
```

Keep the preproduction status visible in deployment reviews. The presence of
full-stack features does not change the beta's production-readiness warning.

## Package imports

The beta packages are exposed through domain-specific subpaths. Typical imports
include:

| Domain | Import path example |
| --- | --- |
| Router | `remix/router` |
| Route trees | `remix/routes` |
| Authentication middleware | `remix/middleware/auth` |
| PostgreSQL data adapter | `remix/data-table/postgres` |
| S3 file storage | `remix/file-storage/s3` |
| Redis session storage | `remix/session-storage/redis` |

Do not import from the `remix` root. One-to-one aliases corresponding to the
older `@remix-run/*` package layout exist only to aid beta migration and are
planned for removal before stable.

UI imports have their own consolidation rules:

- use `remix/ui` for the component runtime;
- use the JSX and server runtime subpaths below `remix/ui`;
- use component paths such as `remix/ui/accordion` and `remix/ui/menu`; and
- use `/primitives` variants where the primitive API is required.

## Generated application layout

Generated applications and the `remix doctor` and `remix routes` commands now
expect action files under `app/actions`, not controller files under
`app/controllers`.

- Root actions live in `app/actions/controller.tsx`.
- Nested route maps require explicit `router.map(...)` calls in
  `app/router.ts`.
- Middleware attached to a controller applies only to that controller's direct
  actions; it does not automatically wrap nested controllers or route groups.

When updating a generated project, move the files and then inspect the route
map. A directory rename alone can leave nested actions unmapped.

## Install a repository preview

With pnpm 9 or newer, install the built `preview/main` branch either for the
full framework workspace or for an individual package workspace:

```sh
pnpm install "remix-run/remix#preview/main&path:packages/remix"
pnpm install "remix-run/remix#preview/main&path:packages/fetch-router"
```

These references select built repository output at a workspace path. They are
appropriate for bleeding-edge evaluation, not as evidence that a beta is ready
for production.

## Migration sequence

1. Identify React Router framework mode versus Remix 3 before changing APIs.
2. If the application is React-based, evaluate React Router v7's incremental
   server-component support before considering a framework rewrite.
3. For a Remix 3 prototype, scaffold from `next` or install the needed
   `preview/main` workspace.
4. Replace the root export and older package aliases with domain subpaths.
5. Move generated controllers to actions and restore explicit nested mappings.
6. Apply the routing, UI, data, and runtime breaking changes in the other
   references before running the project.
