# Platform and Routing

## Runtime and default HTTP adapter

NestJS 11 requires Node.js 20 or newer and no longer supports Node.js 16 or 18.
The default Express platform integration uses Express 5. These requirements are
part of the 11.0.0 changes.

Before upgrading, align local development, CI, container images, and deployment
runtimes with Node.js 20 or newer. Review code and middleware that relies on
Express-specific behavior against Express 5.

## Fastify adapter migration

The following adapter changes belong to the 11.0-migration guidance.

### CORS method allowlist

With `@nestjs/platform-fastify` v11 and Fastify v5, CORS permits only safelisted
methods by default. Explicitly list application methods such as `PUT`, `PATCH`,
and `DELETE` when clients need them:

```typescript
app.enableCors({
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
});
```

Without an explicit list, a route can exist while browser preflight behavior
still prevents calls using a non-safelisted method.

### Middleware matching syntax

Nest middleware matching now uses the latest `path-to-regexp` syntax. An
anonymous `(.*)` wildcard must become a named wildcard such as `*splat`:

```typescript
consumer.apply(ApiMiddleware).forRoutes('*splat');
```

This rule concerns middleware paths passed through Nest's matching layer.
Ordinary Fastify route wildcards are unchanged and should not be mechanically
rewritten as part of this migration.

## Global middleware order

Global-module middleware runs before middleware from imported modules under the
11.0-migration behavior. The position of the global module in the dependency
graph does not change this precedence.

If middleware ordering is observable—for example, one middleware expects state
created by another—test the global middleware first, then the imported-module
middleware. Do not rely on import placement to reverse the order.

## HTTP method recognition

The common, core, and Fastify packages consistently recognize WebDAV HTTP
methods as of 11.0.0. WebDAV methods can participate in Nest routing rather than
being rejected as unknown methods.

## Built-in date transformation

`ParseDatePipe` is exported from `@nestjs/common` as of 11.0.0. Apply it to an
incoming parameter when the handler should receive a `Date`:

```typescript
import { ParseDatePipe, Query } from '@nestjs/common';

find(@Query('since', ParseDatePipe) since: Date) {}
```

The pipe performs the inbound transformation, so the parameter type and runtime
value can both be `Date`.

## Platform migration checklist

- Run all application environments on Node.js 20 or newer.
- Account for Express 5 in default-adapter applications.
- Enumerate every required CORS method for Fastify.
- Convert middleware `(.*)` patterns to named wildcards.
- Leave normal Fastify route wildcards alone.
- Test global middleware before imported-module middleware.
- Route WebDAV methods through the recognized method set where needed.
- Replace custom date parsing with `ParseDatePipe` when its behavior fits.

