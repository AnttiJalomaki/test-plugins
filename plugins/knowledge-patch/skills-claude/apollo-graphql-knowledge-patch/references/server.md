# Apollo Server runtime, HTTP, and execution

## Runtime and integration migration (`server-v5-migration`)

Apollo Server 5 requires Node.js 20.0.0+ and `graphql` 16.11.0+. Upgrade both before
the server. Node.js 24+ is preferable where outgoing requests must honor HTTP proxy
environment variables.

The published JavaScript targets ES2023 (`5.0.0`), so consuming runtimes, bundlers,
and test tools must support ES2023 syntax and built-ins.

### Express integrations

Express middleware is no longer exported from `@apollo/server/express4`. Install
the integration matching the Express major. These packages also support Server 4,
so this step can precede the server upgrade.

```sh
npm install @as-integrations/express4
```

```ts
import { expressMiddleware } from "@as-integrations/express4";
// Use @as-integrations/express5 with Express 5.
```

`startStandaloneServer` now runs directly on Node's HTTP server. Express-specific
behavior such as `x-powered-by`, dynamic `etag` headers, and access to Express APIs
is absent. Use an explicit integration where those behaviors are part of the
contract.

### Built-in `fetch` and proxies

Usage reporting, schema reporting, and subscription callback plugins use Node's
built-in `fetch`, not `node-fetch`. Existing `global-agent` configuration no
longer affects these requests.

- On Node.js 24+, remove `global-agent`, set `NODE_USE_ENV_PROXY=1`, and replace
  `GLOBAL_AGENT_HTTP_PROXY`, `GLOBAL_AGENT_HTTPS_PROXY`, and
  `GLOBAL_AGENT_NO_PROXY` with `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY`.
- On Node.js 20 or 22, install Undici's `EnvHttpProxyAgent` as the global
  dispatcher.
- To retain old behavior, install `node-fetch@2` and pass it to every enabled
  reporting or callback plugin, including plugins implicitly enabled by
  environment variables.

```ts
import nodeFetchFetcher from "node-fetch";

ApolloServerPluginUsageReporting({ fetcher: nodeFetchFetcher });
ApolloServerPluginSchemaReporting({ fetcher: nodeFetchFetcher });
```

### Removed and testing options

Landing-page plugins no longer accept unsafe `precomputedNonce`; delete it.

`@apollo/server-integration-testsuite` no longer contributes DOM globals. Add
`"dom"` to the integration test project's own `tsconfig.json`
`compilerOptions.lib` if required.

```json
{
  "compilerOptions": {
    "lib": ["dom"]
  }
}
```

## Request and validation behavior

### Variable coercion status

`status400ForVariableCoercionErrors` defaults to `true`, so invalid variable values
return HTTP 400. A temporary Server 4 compatibility setting exists:

```ts
new ApolloServer({
  typeDefs,
  resolvers,
  status400ForVariableCoercionErrors: false,
});
```

### Execution and validation ceilings

Server `5.3.0` exposes the GraphQL coercion-error limit through
`executionOptions.maxCoercionErrors`:

```js
new ApolloServer({
  typeDefs,
  resolvers,
  executionOptions: { maxCoercionErrors: 50 },
});
```

It exposes the validation-error limit through `validationOptions.maxErrors`:

```js
new ApolloServer({
  typeDefs,
  resolvers,
  validationOptions: { maxErrors: 10 },
});
```

## Incremental execution protocols

Protocol support depends on the exact Server and GraphQL.js combination.

### Initial Server 5 protocol

The Server 5 migration release enables `@defer` and `@stream` only with exactly
`graphql@17.0.0-alpha.2` and:

```http
Accept: multipart/mixed; deferSpec=20220824
```

GraphQL.js 16, later v17 alphas, and a later final v17 do not enable incremental
delivery under that release.

### Alpha 9 and protocol v0.2 (`5.1.0`)

Server 5.1 uses `graphql@17.0.0-alpha.9` and:

```http
Accept: multipart/mixed; incrementalSpec=v0.2
```

`graphql@16` remains supported without incremental delivery. Deployments using
alpha.2 must upgrade for this protocol. Legacy
`multipart/mixed; deferSpec=20220824` acceptance requires
`@yaacovcr/transform`; without it, the server rejects the header.

Unsuffixed legacy incremental result types acquire an `Alpha2` suffix. Alpha 9
types use `Alpha9` and include completed and pending result types.

```ts
import type {
  GraphQLExperimentalFormattedInitialIncrementalExecutionResultAlpha9,
  GraphQLExperimentalFormattedCompletedResultAlpha9,
  GraphQLExperimentalPendingResultAlpha9,
} from "@apollo/server";
```

### Explicit legacy executor (`5.2.0`)

Installing the transform is no longer sufficient. Pass its compatibility executor
to `ApolloServer`, or legacy accept headers fail:

```ts
import { legacyExecuteIncrementally } from "@yaacovcr/transform";

const server = new ApolloServer({
  legacyExperimentalExecuteIncrementally: legacyExecuteIncrementally,
});
```

## Standalone HTTP hardening

### Request-body encodings (`5.4.0`)

`startStandaloneServer` accepts only UTF-8, UTF-16 LE/BE, and UTF-32 LE/BE request
bodies. Other character sets return `415 Unsupported Media Type`, closing a
denial-of-service vector. Other integrations are unaffected.

### GET `Content-Type` (`5.5.0`)

`@apollo/server/standalone` rejects GraphQL GET requests whose `Content-Type` is
not `application/json` with optional parameters, returning HTTP 415. An omitted
header remains allowed. With default CSRF prevention, a headerless request still
needs a non-empty `X-Apollo-Operation-Name` or `Apollo-Require-Preflight`. This is
especially relevant when authentication uses cookies or HTTP Basic Auth.
