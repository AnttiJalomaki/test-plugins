---
name: aspnet-core-knowledge-patch
description: ASP.NET Core
version: 11.0 Preview 6
license: MIT
metadata:
  author: Nevaberry
---


# ASP.NET Core Knowledge Patch

Use this skill when implementing, migrating, reviewing, or debugging ASP.NET
Core applications. Inspect the app's target framework, package versions,
render modes, hosting stack, and generated templates before applying guidance.
For preview APIs, confirm that the project uses the matching preview and prefer
the newest API name described here.

## Reference Index

| Reference | Topics |
| --- | --- |
| [components-navigation-and-interop.md](references/components-navigation-and-interop.md) | Blazor rendering, routing, circuits, virtualization, JavaScript interop, culture, browser startup |
| [forms-validation-and-persistence.md](references/forms-validation-and-persistence.md) | Blazor and Minimal API validation, labels, TempData, session, SSR forms |
| [hosting-http-caching-and-security.md](references/hosting-http-caching-and-security.md) | Kestrel, HTTP.sys, compression, output caching, CSRF, certificates, memory pools |
| [migrations-assets-and-tooling.md](references/migrations-assets-and-tooling.md) | Upgrade removals, static assets, WebAssembly Gateway, templates, workers, tests |
| [observability-identity-and-signalr.md](references/observability-identity-and-signalr.md) | OpenTelemetry, diagnostics, authentication metrics, passkeys, SignalR, development JWTs |
| [openapi-minimal-apis-and-json.md](references/openapi-minimal-apis-and-json.md) | OpenAPI 3.1/3.2, transformers, schemas, Minimal API binding, JSON parsing, unions |

## Working Rules

1. Identify whether the project is stable or preview before copying an API.
2. Treat later preview renames and defaults as authoritative for later previews.
3. Preserve explicit compatibility switches only as temporary migration aids.
4. Check render mode before applying Blazor guidance; static SSR, Interactive
   Server, and WebAssembly do not share every behavior.
5. Check OpenAPI document version and `Microsoft.OpenApi` package generation
   before editing transformers.
6. Test proxy, cache, antiforgery, compression, and request-target changes at
   the HTTP boundary rather than only through application-level unit tests.
7. Load the narrow reference file for implementation details and edge cases.

## Breaking and Migration Quick Reference

### Remove obsolete Blazor boot assumptions

- Do not read or patch `blazor.boot.json`; boot configuration is in
  `dotnet.js`, and direct-file customization has no documented replacement.
- Remove `BlazorCacheBootResources`; fingerprinted browser caching replaced
  the custom boot-resource cache.
- Select a standalone WebAssembly environment with
  `WasmApplicationEnvironmentName` in the project file.
- Do not depend on the `Blazor-Environment` header, launch settings, or
  `ASPNETCORE_ENVIRONMENT` for that selection.

### Update Identity navigation during migration

When enabling `BlazorDisableThrowNavigationException` in an older Individual
Accounts app, remove the explicit `InvalidOperationException` from
`IdentityRedirectManager.RedirectTo` and remove its five `DoesNotReturn`
annotations. Follow the dedicated migration path when adding passkeys to an
existing app.

### Update routing behavior deliberately

- Replace the removed router `<NotFound>` fragment with
  `Router.NotFoundPage`, `NavigationManager.NotFound()`, and optionally
  `NavigationManager.OnNotFound`.
- `NavLinkMatch.All` compares only the path. Use its AppContext compatibility
  switch only if query-string and fragment matching is required.
- Same-page `NavigateTo` calls retain scroll position.
- For relative links resolved from the current page, set
  `NavigationOptions.RelativeToCurrentUri` or the matching `NavLink` property.

### Account for WebAssembly response streaming

WebAssembly response streaming is enabled by default. `ReadAsStreamAsync`
returns `BrowserHttpReadStream`, which cannot perform synchronous reads. Opt
out per request with `SetBrowserResponseStreamingEnabled(false)`, or use the
project property or environment switch described in the components reference.

### Migrate OpenAPI transformers

- OpenAPI entities use interfaces plus inline and reference implementations.
- Replace `OpenApiSchema.Nullable` checks with `JsonSchemaType.Null`.
- Replace `OpenApiAny` values with `JsonNode`.
- Expect `Microsoft.OpenApi` dependency upgrades to require integration
  changes even when emitting an older document version.
- Later previews generate OpenAPI 3.2 by default; pin an earlier
  `OpenApiVersion` when downstream tooling requires it.

### Make custom JSON converters sequence-safe

ASP.NET Core JSON input now commonly uses `PipeReader`. A converter must read
`Utf8JsonReader.ValueSequence` when `HasValueSequence` is true instead of
assuming `ValueSpan` contains the entire token. The stream-parsing AppContext
switch is only a temporary fallback.

### Adopt preview API renames

- Use `EnvironmentView`, not the earlier `EnvironmentBoundary` name.
- Use `NavigationManager.GetUriWithFragment`, not `GetUriWithHash`.
- Use `WithBrowserOptions`, not `WithBrowserConfiguration`.
- Use `PreserveDom` with positive semantics and the `TimeSpan`-valued
  `CircuitInactivityTimeout`.
- Custom Minimal API validation resolvers must use the specialized type,
  parameter, and property interfaces and call
  `ValidateContext.AddValidationError`.

## Security and Hosting Quick Reference

### Review automatic cross-origin CSRF rejection

Apps built with `WebApplication.CreateBuilder` reject unsafe cross-origin
browser form requests using Fetch Metadata and `Origin` information. Blazor
Web Apps no longer need an explicit `UseAntiforgery()` call. Use endpoint or
application opt-outs only after confirming the cross-origin trust model, or
replace it with `ICsrfProtection`.

### Handle changed exception diagnostics

Handled exceptions no longer emit diagnostics by default. Configure
`ExceptionHandlerOptions.SuppressDiagnosticsCallback` when handled failures
must still be logged or traced.

### Check compression and cache correctness

Zstandard is enabled by the standard response-compression and
request-decompression middleware. Tune its quality from 1 through 22.
Compression middleware now emits `Vary: Accept-Encoding` even when a response
is not compressed, preventing shared-cache variant confusion.

### Preserve request-target and timeout protections

- Kestrel preserves `%2F` in absolute-form HTTP/1.1 paths.
- `RequestHeadersTimeout` applies to incomplete HTTP/2 and HTTP/3 trailer
  header blocks.
- Register `UseTlsClientHelloListener` before `UseHttps`; the older TLS Client
  Hello callback property is obsolete.
- An HTTP.sys request-queue security descriptor applies only when a new queue
  is created.

## Components and Forms Quick Reference

### Use source-generated recursive validation

Call `AddValidation`, place the root model in a C# file, and annotate it with
`ValidatableType`. Use `SkipValidation` where traversal must stop. Both the app
and a model-providing assembly must register validation when models cross
assembly boundaries.

### Await asynchronous validation

Blazor `EditForm` awaits tasks registered through
`EditContext.AddValidationTask`; use `ValidateAsync` and observe pending,
faulted, cancellation, and supersession behavior. Minimal APIs also execute
the asynchronous DataAnnotations contracts after `AddValidation`.

### Distinguish SSR persistence scopes

- `[PersistentState]` persists prerendered component or service state.
- `[SupplyParameterFromTempData]` uses one-time TempData semantics.
- `[SupplyParameterFromSession]` uses configured HTTP session and JSON
  serialization.
- TempData is cascaded as `ITempData` during server-side rendering.

### Configure virtualization by user intent

`Virtualize<TItem>` measures variable-height items, anchors viewport edges,
and can start or scroll to an index. Use `ItemComparer` for refreshed
reference objects, choose `AnchorMode.End` for append-following experiences,
and call `ScrollToIndexAsync` only after the first interactive render.

### Configure browser startup from the server

Use `WithBrowserOptions` to serialize Server, WebAssembly, Auto, logging,
reconnection, SSR DOM-preservation, and environment options from C#.
JavaScript startup also accepts unified nested `circuit` and `webAssembly`
objects.

## API and Observability Quick Reference

### Generate accurate OpenAPI

- Documentation XML can populate endpoint summaries, remarks, parameters,
  returns, and referenced-project comments.
- Advertise concrete file-result types to generate binary response schemas.
- `QUERY` operations are native in 3.2 and use
  `x-oai-additionalOperations` in 3.0 and 3.1.
- Multiple response declarations for one status code are preserved by media
  type or `anyOf`.
- C# unions generate `anyOf` without a discriminator and are JSON-only in the
  supported ASP.NET Core surfaces.

### Collect framework-native tracing

Subscribe OpenTelemetry to the `Microsoft.AspNetCore` activity source.
Framework request activities already carry HTTP server semantic-convention
attributes; suppress them only with the dedicated AppContext switch.

### Refresh SignalR authentication intentionally

Enable authentication refresh on the mapped hub and configure the .NET client
with `WithAuthenticationRefresh`. Cancellation of a regular .NET
`InvokeAsync` now reaches the hub method's `CancellationToken`.

