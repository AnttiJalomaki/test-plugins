---
name: apollo-graphql-knowledge-patch
description: Apollo GraphQL
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Apollo GraphQL

Use this skill when changing Apollo Client, Apollo Server, or Apollo Router code and
configuration. First identify the product and installed version from package
manifests, lockfiles, Router binaries, images, or Helm values. Then open only the
references relevant to the task.

Project code, schemas, tests, and runtime behavior remain authoritative. Pay special
attention to exact minor releases: incremental-delivery protocols, response-cache
namespaces, rate-limit status codes, and telemetry settings changed more than once.

## Reference index

| Reference | Topics |
| --- | --- |
| `references/client-migration.md` | Client 4 package boundaries, constructor and link migration, RxJS, errors, types, hooks, React, SSR, testing |
| `references/client-data-and-cache.md` | Result states, fragments, cache behavior, mutations, incremental delivery, refetch events, client awareness |
| `references/server.md` | Server 5 runtime and integration migration, HTTP behavior, incremental execution, execution and validation limits |
| `references/router-migration-and-security.md` | Router v2 migration, configuration, plugins, authentication, CORS, HTTP hardening, limits, authorization |
| `references/router-connectors-and-extensions.md` | Connectors, coprocessors, Rhai, custom plugins, headers, TLS, Unix sockets |
| `references/router-caching-and-traffic.md` | Persisted queries, response/entity/query-plan caches, Redis, traffic shaping, subscriptions, reloads |
| `references/router-execution-and-delivery.md` | Query planning, demand control, GraphQL correctness, federation, errors, incremental and subscription delivery |
| `references/router-observability.md` | OpenTelemetry migration, metrics, spans, selectors, exporters, GraphOS reporting, logging |

## Breaking migrations and deprecations

### Apollo Client 4

- Run the Client 3-to-4 codemod, then repair public imports. React APIs live under
  `@apollo/client/react`, React test helpers under
  `@apollo/client/testing/react`, and `rxjs` is a peer dependency.
- Replace constructor transport shortcuts with an explicit `HttpLink`; move
  awareness, devtools, and cache-priority settings to their new nested options.
- Treat returned `error` as the unified failure channel. `ApolloError` and
  `result.errors` are gone; use error-class `.is` guards.
- Account for `errorPolicy` on network, GraphQL, protocol, query, mutation, and
  subscription failures. Watched-query failures normally arrive through `next`.
- Move observable operators to RxJS. Wrap an `ObservableQuery` with `from(...)`
  where an RxJS `Observable` is required.
- Do not rely on option changes to execute `useLazyQuery`; invoke its execute
  function, handle `AbortError`, and retain the promise only when completion after
  unmount or a newer execution is intentional.
- Use class-based links. `SetContextLink` receives
  `(previousContext, operation)`, and link composition requires `ApolloLink`
  instances.
- Configure a local-state subsystem for `@client`, and make custom caches implement
  fragment matching.
- Replace render-prop components, HOCs, and `ApolloConsumer` with hooks. Replace
  legacy SSR tree walkers with `prerenderStatic`.
- Read `dataState` before assuming data is complete, particularly with partial or
  deferred results.

### Apollo Server 5

- Upgrade to Node.js 20+ and `graphql` 16.11.0+; ensure consumers support ES2023.
- Import Express middleware from `@as-integrations/express4` or
  `@as-integrations/express5`. The standalone server no longer embeds Express.
- Reporting plugins use built-in `fetch`; migrate proxy configuration or inject a
  fetcher explicitly.
- Invalid variable coercion defaults to HTTP 400. Preserve the former behavior only
  with the temporary compatibility option.
- Match the installed GraphQL.js alpha, server release, executor option, and client
  `Accept` header when enabling incremental delivery.
- Remove landing-page `precomputedNonce`; add DOM libs explicitly to integration
  test projects that need them.
- Expect standalone HTTP hardening for body encodings and GET `Content-Type`.

### Apollo Router 2

- Materialize a major configuration upgrade before deployment with
  `router config upgrade`; v2 does not migrate v1 configuration at startup.
- Replace `router --schema` with `router config schema`.
- Expect busy routers to reject work instead of building an in-memory queue.
  Revisit timeouts, concurrency, rate limits, and alerts for `503`/`504`.
- Migrate Router-specific and underscore-named metrics to OpenTelemetry and dotted
  instrument names before rollout.
- Move Jaeger and Zipkin collection to OTLP endpoints.
- Update namespaced request-context keys everywhere: plugins, Rhai, coprocessors,
  and telemetry selectors.
- Update Rust plugin checkpoint, layer, lock, harness, and initialization APIs.
  Services are built once and cloned per request.
- Convert `:parameter` routes to `{parameter}`, root body paths with `$`, and
  conditional logging to telemetry events.
- Remove remote-supergraph polling flags; remote URL sources do not hot-reload.
- Remove deprecated Router configuration keys:
  `traffic_shaping.deduplicate_variables`,
  `connectors.preview_connect_v0_4`, and
  `persisted_queries.experimental_local_manifests`.

## Security-sensitive defaults

- Keep Router introspection depth limiting enabled unless a known introspection
  workload requires an exception.
- Treat invalid Router CORS values as startup failures.
- Validate JWT issuer, audience, expiry, and claim types deliberately. Use
  `on_error: Continue` or `allow_missing_exp` only as explicit policy choices.
- Upgrade past the Router 2.1 resource-exhaustion fixes; if temporarily pinned
  earlier, require persisted-query IDs with safelist enforcement.
- Prefer subgraph-error extension allowlists over denylists.
- Expect strict input-object variable validation and recursive-selection checks.
  Measurement or warning modes are migration aids, not equivalent enforcement.
- Router and standalone Server reject incompatible GET content types with HTTP 415;
  headerless requests still pass through CSRF rules.
- Sensitive Router headers are masked by default. Copying secrets into coprocessor
  bodies or context can move them outside automatic masking.
- Root-type authorization directives compose onto the shared supergraph root; put
  them on fields when authorization must stay local to one contribution.

## High-use current features

### Client

- Use `useSuspenseFragment` when an incomplete fragment should suspend.
- Select an incremental handler that exactly matches the server protocol.
- Declare default options and signature style so inferred query and mutation types
  reflect runtime `errorPolicy` and partial-data behavior.
- Use `RefetchEventManager` for focus, online, imperative, or custom event-driven
  refetching.
- Use fragment arrays for index-aligned reads and `from: null` for a completed null
  fragment source.

### Server

- Configure coercion and validation ceilings through `executionOptions` and
  `validationOptions`.
- Choose the modern incremental protocol unless an explicitly configured legacy
  executor is required.
- Use an explicit web-framework integration when middleware or Express behavior is
  part of the application contract.

### Router

- Prefer the GA `response_cache` namespace and configure Redis, invalidation
  indexes, cache identity, TTL fallbacks, and client cache headers separately.
- Use Connector source-level traffic shaping, TLS, headers, response limits, and
  telemetry where different REST origins have different operational policies.
- Use per-subgraph demand-control ceilings to skip only over-budget subgraph work
  while allowing the rest of an operation to continue.
- Correlate spans, logs, and events with `context_id`; monitor Router overhead,
  request allocation, connection acquisition, pipeline count, and cardinality
  overflow.
- Configure per-exporter sampling under the common sampler ceiling.
- Bound subscription lifetime and deduplicate only when ignored headers or JWT
  context cannot change event personalization.
- Use retrying reloads for transient artifact or schema failures, and mutable OCI
  tags when tag promotion is the intended deployment workflow.

## Working procedure

1. Identify Client, Server, Router, or a combination.
2. Determine exact versions and protocol peers.
3. Read the indexed topic references for the requested behavior.
4. Search the project for removed names and deprecated keys before editing.
5. Make migrations explicit in code or checked-in configuration.
6. Test both successful and failure paths, including HTTP status and error policy.
7. Validate telemetry names and labels against dashboards and alert rules.
8. For cache changes, plan for key regeneration and mixed-version rollouts.
9. For incremental or subscription changes, test first, intermediate, terminal, and
   disconnect payloads.
10. For security compatibility switches, document why enforcement is relaxed and
    how the switch will be removed.
