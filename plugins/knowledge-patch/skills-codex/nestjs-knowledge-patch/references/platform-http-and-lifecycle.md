# Platform, HTTP, and Lifecycle

Use this reference for runtime upgrades, HTTP adapters, middleware, lifecycle
hooks, and protocol-facing utilities. It incorporates the `11.0-migration` and
`11.0.0` guidance.

## Runtime and default HTTP platform

NestJS 11 requires Node.js 20 or newer; Node.js 16 and 18 are no longer
supported. Align developer machines, CI images, container bases, and production
runtimes before debugging framework-level failures.

The default Express platform integration uses Express 5. Validate route
matching and middleware behavior as part of the platform migration.

## Fastify v5 migration

### CORS method defaults

With `@nestjs/platform-fastify` v11 and Fastify v5, CORS allows only safelisted
methods by default. Applications that accept methods such as `PUT`, `PATCH`, or
`DELETE` must list them explicitly:

```typescript
app.enableCors({
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
});
```

Audit preflight requests as well as the controller routes themselves.

### Middleware path matching

Nest middleware matching uses the latest `path-to-regexp`. Replace the old
anonymous `(.*)` matcher with a named wildcard:

```typescript
consumer.apply(ApiMiddleware).forRoutes('*splat');
```

This is a middleware-matcher migration. Ordinary Fastify route wildcards have
not changed, so do not rewrite them solely because of this rule.

## Middleware from global modules

Middleware registered by global modules runs before middleware from imported
modules. This ordering does not depend on the global module's position in the
dependency graph.

Review authentication, request context, tracing, and mutation middleware for
order dependencies. Tests should assert observable order when imported-module
middleware expects state established by another layer.

## Initialization and termination

Termination hooks run in the reverse order of initialization. Given
dependencies `A -> B -> C`:

- Initialization runs `C -> B -> A`.
- `OnModuleDestroy` runs `A -> B -> C`.
- `BeforeApplicationShutdown` runs `A -> B -> C`.
- `OnApplicationShutdown` runs `A -> B -> C`.

Global modules initialize first and are destroyed last. Cleanup logic should
not assume the earlier destruction sequence; explicitly test providers that
close shared connections or depend on other providers during teardown.

## HTTP method recognition

WebDAV HTTP methods are recognized consistently by the common, core, and
Fastify packages. They can participate in Nest routing instead of being treated
as unknown methods.

## WebSocket extension points

WebSocket errors can retain a cause, allowing the original failure to remain
attached when a higher-level error is created.

The `ws` adapter exposes a message-parser extension point. Use it when the
application needs a custom wire format while retaining the Nest WebSocket
adapter.

## Date parsing

`ParseDatePipe` is exported by `@nestjs/common` and converts an incoming
parameter into a `Date`:

```typescript
import { ParseDatePipe, Query } from '@nestjs/common';

find(@Query('since', ParseDatePipe) since: Date) {}
```

Use the transformed `Date` type in the handler rather than manually reparsing
the original string.

