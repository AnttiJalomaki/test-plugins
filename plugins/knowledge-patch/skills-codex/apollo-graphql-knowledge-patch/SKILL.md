---
name: apollo-graphql-knowledge-patch
description: Apollo GraphQL
version: null
license: MIT
metadata:
  author: Nevaberry
---


# Apollo GraphQL Knowledge Patch

Load this skill when changing, upgrading, debugging, or reviewing Apollo Client,
Apollo Server, Apollo Router, Connectors, coprocessors, caching, demand control,
subscriptions, or Apollo telemetry.

## How to use this patch

1. Identify the product and exact installed version from package manifests,
   Router binaries, images, Helm values, and lockfiles.
2. Open the reference file for the task. Product release lines evolve
   independently; never transfer a Client behavior to Server or Router.
3. Apply an entry only when its inline version attribution is relevant to the
   installed release.
4. Prefer the repository's code, schema, configuration, and tests when they
   demonstrate behavior more specific than this guidance.
5. During an upgrade, review every crossed migration and deprecation entry,
   then validate HTTP behavior, error shapes, cache semantics, and telemetry.

## Reference index

| Reference | Topics |
| --- | --- |
| [client-react.md](references/client-react.md) | Client migration, React hooks, links, errors, cache, SSR, testing, refetch events |
| [server-runtime.md](references/server-runtime.md) | Server runtime and integrations, HTTP semantics, incremental delivery, execution limits |
| [router-configuration-security.md](references/router-configuration-security.md) | Router v2 migration, configuration, authentication, CORS, limits, networking, deployment |
| [connectors-extensions.md](references/connectors-extensions.md) | Connectors, coprocessors, Rhai, Rust plugins, context, header handling |
| [caching-persisted-queries.md](references/caching-persisted-queries.md) | Response and entity caches, Redis, persisted queries, reloads |
| [planning-execution-demand.md](references/planning-execution-demand.md) | Query planning, demand control, validation, coercion, authorization |
| [subscriptions-transport.md](references/subscriptions-transport.md) | Subscriptions, WebSockets, multipart transport, deduplication, lifecycle |
| [telemetry-observability.md](references/telemetry-observability.md) | Metrics, traces, selectors, exporters, logging, error reporting |

## Breaking migrations first

### Apollo Client 4

- Install `rxjs`; import React APIs from `@apollo/client/react` and
  `MockedProvider` from `@apollo/client/testing/react`.
- Run the Client migration codemod, then remove private `.js` and `.cjs`
  imports.
- Construct an `HttpLink`; constructor shortcuts such as `uri`, `headers`, and
  `credentials` are gone.
- Replace `ApolloError` and the separate `errors` result with the unified
  `error` value and the supplied error-class guards.
- Treat `ObservableQuery` as a `Subscribable`; wrap it with RxJS `from()` when
  an `Observable` is required.
- Move context typing to `DefaultContext` declaration merging and use
  API-namespaced types.
- Configure an incremental handler before using `@defer` or `@stream`.
- Replace render-prop components, HOCs, and `ApolloConsumer` with hooks and
  `useApolloClient`.

```ts
import { ApolloClient, HttpLink, InMemoryCache } from "@apollo/client";

const client = new ApolloClient({
  cache: new InMemoryCache(),
  link: new HttpLink({ uri: "/graphql" }),
});
```

### Apollo Server 5

- Upgrade to Node.js 20 or newer and `graphql` 16.11.0 or newer.
- Install `@as-integrations/express4` or `@as-integrations/express5`; the
  Express middleware is no longer exported from `@apollo/server/express4`.
- Audit proxy behavior because reporting plugins use the Node.js built-in
  `fetch`.
- Expect invalid variable coercion to return HTTP 400 by default.
- Do not depend on Express behavior in `startStandaloneServer`.
- Match the exact GraphQL.js incremental format, client `Accept` header, and
  compatibility executor described in the Server reference.

### Apollo Router 2

- Materialize major-version configuration upgrades before deployment; Router
  v2 does not apply them while loading configuration.
- Replace removed and renamed Router metrics with their OpenTelemetry
  equivalents before rollout.
- Rename request context keys, endpoint path parameters, header-propagation
  JSONPaths, and telemetry selectors.
- Rework Rust plugin services for a pipeline built once and cloned per request.
- Expect overload to reject work instead of accumulating it in memory.
- Validate CORS and introspection settings because invalid or newly restricted
  input can now prevent startup or return a different HTTP status.

```bash
router config upgrade --diff router.yaml
router config upgrade router.yaml > router.next.yaml
router config schema
```

## High-value deprecations

### Client

- Avoid `useQuery` and `useLazyQuery` lifecycle callbacks.
- Replace `useMutation({ ignoreResults: true })` with `client.mutate`.
- Replace `queryRef.toPromise()` with `preloadQuery.toPromise(queryRef)`.
- Replace `ObservableQuery.setOptions()` with `reobserve()` and `result()` with
  `firstValueFrom(from(observable))`.
- Remove `addTypename` configuration, `canonizeResults`, legacy link creator
  functions, and removed component APIs as migration reaches Client 4.

### Router

- Remove `traffic_shaping.deduplicate_variables`; variable deduplication is
  always enabled.
- Rename `connectors.subgraphs` to `connectors.sources`.
- Rename `persisted_queries.experimental_local_manifests` to
  `persisted_queries.local_manifests`.
- Replace the native Zipkin exporter with Zipkin's OTLP endpoint.
- Replace Router OpenTelemetry header helpers with the corresponding
  `opentelemetry_http` types.
- Move response caching from `preview_response_cache` to `response_cache`.

## Error and HTTP compatibility

- Client error policies govern GraphQL, protocol, and network failures. Under
  Client 4, watched queries normally deliver failures through `next` results.
- A `useMutation` completion callback that throws rejects the mutation promise;
  the failure does not flow into that mutation's `onError`.
- Server 5 defaults variable-coercion failures to HTTP 400.
- Standalone Server and Router reject unsupported `GET` request content types
  with HTTP 415 in their hardened releases.
- Router result-coercion validation can convert mismatched or missing response
  fields into `RESPONSE_VALIDATION_FAILED`.
- Router rate-limit status changed more than once; select retry behavior from
  the exact installed Router version.
- Response-size limits abort oversized downstream bodies with
  `SUBREQUEST_HTTP_ERROR`.

## Cache and persisted-query essentials

- Treat response-cache and entity-cache key changes as cold-cache events during
  rollout.
- A subgraph's `Cache-Control: max-age` wins over configured fallback TTL.
- GraphQL errors make cacheable Router responses `no-store`.
- Distinguish `no-store` from `no-cache`; the Router does not implement the
  revalidation implied by `no-cache`.
- Configure invalidation indexes deliberately. Re-enabling an index does not
  backfill entries written while it was disabled.
- Use the GA `local_manifests` key and enable `hot_reload` when local persisted
  query files must refresh independently of Router hot reload.

## Connectors and extension points

- Key per-source Connector settings as `<subgraph>.<source>`.
- Router YAML header rules override headers declared by Connector directives.
- Link the schema to the intended Connector specification; parser and mapping
  semantics differ between v0.2/v0.3 and v0.4.
- Use selective coprocessor bodies and context keys to minimize exposure and
  avoid accidental mutation of unrelated response fields.
- Remember that sensitive-header masking does not protect a secret copied into
  a coprocessor body or context.
- Use OpenTelemetry instruments from the Router meter provider in Rust
  plugins; tracing-field metric shims are gone.

## Planning, validation, and demand control

- Use cooperative query-planning cancellation in `measure` mode before
  enforcing time or memory ceilings.
- Size histogram buckets beyond configured timeouts or long operations will
  collapse into the highest bucket.
- Per-subgraph demand limits null only the skipped subgraph fetches while the
  rest of the operation continues.
- Strict input-object validation rejects unknown variable fields; use its
  measurement mode only as a temporary compatibility step.
- Root-type authorization directives compose onto the shared supergraph root.
  Put a policy on individual root fields when it should remain local.

## Subscriptions and incremental transport

- Subscription deduplication is enabled by default in Client 4 and can also be
  configured in the Router. Audit headers and authentication context before
  sharing a transport.
- Do not ignore JWT context for personalized subscription streams.
- Set `subscription.max_lifetime` when long-lived connections need a bounded
  lifecycle.
- Match incremental protocol handlers and `Accept` headers across Client,
  Server, Router, gateways, and tests.
- For multipart responses, query deduplication remains active until the final
  chunk and Client streaming results remain loading until completion.

## Telemetry checklist

- Use dotted Router metric names and OpenTelemetry semantic instruments.
- Put static metric attributes in the resource and dynamic attributes on
  instruments.
- Treat selector JSONPath roots as local to the selected payload part.
- Remove OTLP endpoint environment variables when the installed Router rejects
  them at startup.
- Set explicit cardinality limits only with memory cost and overflow monitoring
  in mind.
- Configure exporter-specific sampling no higher than the common sampling
  fraction.
- Prefer error-only selectors over full response bodies when recording failure
  details.
- Verify whether Apollo-only instruments are also available to third-party
  exporters before building dashboards around them.

## Verification

After changes, exercise:

- one successful and one failing GraphQL operation;
- variable validation, content-type, CORS, and authentication boundaries;
- cache hit, miss, bypass, invalidation, and schema/config reload paths;
- subscriptions or deferred results through their final frame;
- overload, timeout, rate-limit, and response-size behavior;
- telemetry names, attributes, units, sampling, and cardinality overflow.

Check logs for auto-migration and deprecation warnings. Treat startup validation
errors as migration tasks rather than bypassing them without understanding the
security or observability consequence.
