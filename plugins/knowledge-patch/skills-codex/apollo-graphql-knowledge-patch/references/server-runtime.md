# Apollo Server Runtime and Integrations

## Runtime and build compatibility

The server-v5-migration requires Node.js 20.0.0 or newer and `graphql`
16.11.0 or newer. Node.js 24 or newer is preferable when built-in proxy
environment support is needed.

Apollo Server 5.0.0 is compiled to ES2023 rather than ES2020. Build tools and
runtimes consuming the published JavaScript must understand ES2023 syntax and
built-ins.

## Web framework integrations

The server-v5-migration removes the Express middleware export from
`@apollo/server/express4`. Install the package matching the Express major:

```bash
npm install @as-integrations/express4
```

```ts
import { expressMiddleware } from "@as-integrations/express4";
// Use @as-integrations/express5 with Express 5.
```

These integration packages also work with Apollo Server 4, so the integration
change can be staged before the Server 5 upgrade.

`startStandaloneServer` no longer embeds Express. It runs directly on Node's
HTTP server, so `x-powered-by`, dynamic `etag` behavior, and type-unsafe access
to Express APIs disappear. Use an explicit Express integration when the
application requires Express semantics.

The integration test suite no longer compiles with ambient DOM globals during
the server-v5-migration. Projects using
`@apollo/server-integration-testsuite` that need those types must add `"dom"`
to their own `tsconfig.json` TypeScript `lib`.

## Reporting, callbacks, and proxies

Usage reporting, schema reporting, and subscription callback plugins use the
Node.js built-in `fetch` by default after the server-v5-migration.
`global-agent` therefore no longer affects those requests.

- On Node.js 24 or newer, remove `global-agent`, set
  `NODE_USE_ENV_PROXY=1`, and rename `GLOBAL_AGENT_HTTP_PROXY`,
  `GLOBAL_AGENT_HTTPS_PROXY`, and `GLOBAL_AGENT_NO_PROXY` to `HTTP_PROXY`,
  `HTTPS_PROXY`, and `NO_PROXY`.
- On Node.js 20 or 22, configure Undici's `EnvHttpProxyAgent` as the global
  dispatcher.
- To keep the old behavior, install `node-fetch@2` and pass it explicitly as
  `fetcher` to every enabled usage-reporting, schema-reporting, and subscription
  callback plugin. Include plugins enabled implicitly by environment variables.

## HTTP status and body hardening

### Variable coercion

In the server-v5-migration, `status400ForVariableCoercionErrors` defaults to
`true`, restoring HTTP 400 for invalid variable values. Set it to `false` only
for temporary compatibility with Server 4 behavior; the option is intended for
future removal.

### Standalone body encodings

Apollo Server 5.4.0 restricts `startStandaloneServer` request bodies to UTF-8,
UTF-16 LE or BE, and UTF-32 LE or BE. Other character sets return HTTP 415,
`Unsupported Media Type`, closing a denial-of-service path. Other Server
integrations are unaffected.

### Standalone GET requests

Apollo Server 5.5.0 makes `@apollo/server/standalone` reject a GraphQL `GET`
whose `Content-Type` is not `application/json` with optional parameters,
returning HTTP 415. Omitting `Content-Type` is allowed, but default CSRF
prevention still requires non-empty `X-Apollo-Operation-Name` or
`Apollo-Require-Preflight` on a headerless request. Prioritize this hardening
when cookies or HTTP Basic Auth protect the graph.

## Incremental delivery

### Initial Server 5 format

During the server-v5-migration, incremental delivery is enabled only with
exactly `graphql@17.0.0-alpha.2` and this request header:

```http
Accept: multipart/mixed; deferSpec=20220824
```

GraphQL.js 16, newer v17 alphas, and a later official v17 release do not enable
incremental delivery under that initial combination.

### Alpha 9 and the v0.2 protocol

Apollo Server 5.1.0 moves `@defer` and `@stream` to
`graphql@17.0.0-alpha.9` and the v0.2 incremental protocol:

```http
Accept: multipart/mixed; incrementalSpec=v0.2
```

`graphql@16` remains supported without incremental delivery. Upgrade
alpha.2-based deployments. The old unsuffixed incremental result types receive
an `Alpha2` suffix; the alpha.9 types use `Alpha9` and include completed and
pending result types.

```ts
import type {
  GraphQLExperimentalFormattedInitialIncrementalExecutionResultAlpha9,
  GraphQLExperimentalFormattedCompletedResultAlpha9,
  GraphQLExperimentalPendingResultAlpha9,
} from "@apollo/server";
```

Version 5.1.0 can accept the legacy 2022 header when
`@yaacovcr/transform` is installed. Without it, that header returns an error.

Apollo Server 5.2.0 additionally requires the compatibility executor to be
passed to `ApolloServer`; installing the package alone is insufficient:

```ts
import { legacyExecuteIncrementally } from "@yaacovcr/transform";

const server = new ApolloServer({
  legacyExperimentalExecuteIncrementally: legacyExecuteIncrementally,
});
```

Keep the GraphQL.js alpha, server executor, response types, client handler, and
`Accept` header aligned.

## Execution and validation limits

Apollo Server 5.3.0 exposes GraphQL execution options through
`executionOptions`. Set `maxCoercionErrors` there:

```js
new ApolloServer({
  typeDefs,
  resolvers,
  executionOptions: { maxCoercionErrors: 50 },
});
```

The same release exposes validation options through `validationOptions`; use
`maxErrors` for the validation-error ceiling:

```js
new ApolloServer({
  typeDefs,
  resolvers,
  validationOptions: { maxErrors: 10 },
});
```

## Landing pages

The server-v5-migration removes the unsafe, deprecated
`precomputedNonce` landing-page option. Delete it without replacement; the
Cloudflare Workers compatibility issue it addressed no longer needs a fixed
nonce.
