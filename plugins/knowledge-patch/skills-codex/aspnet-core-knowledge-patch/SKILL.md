---
name: aspnet-core-knowledge-patch
description: ASP.NET Core
version: 11.0 Preview 6
license: MIT
metadata:
  author: Nevaberry
---


# ASP.NET Core Knowledge Patch

Use this skill when upgrading or developing ASP.NET Core applications whose behavior may depend on recent Blazor, Minimal API, OpenAPI, hosting, Kestrel, security, validation, or SignalR changes.

Prefer the application's project files, source, tests, and observed runtime behavior when they disagree with general guidance. Preview APIs may have evolved between previews; use the latest name and behavior documented here unless the project is deliberately pinned to an earlier preview.

## Reference index

| Reference | Topics |
| --- | --- |
| [Blazor components and navigation](references/blazor-components-navigation.md) | Navigation, routing, rendering, metadata components, circuit lifecycle, browser options, virtualization, QuickGrid |
| [Blazor WebAssembly and tooling](references/blazor-wasm-tooling.md) | Standalone migration, static assets, streaming, JS interop, globalization, workers, templates, Gateway proxy |
| [Forms, validation, state, and Identity](references/forms-validation-state-identity.md) | Recursive and async validation, localization, persistent state, TempData, session parameters, passkeys, Identity redirects |
| [OpenAPI and HTTP APIs](references/openapi-http-apis.md) | OpenAPI defaults and transformers, response schemas, QUERY, unions, JSON parsing, endpoint filters, testing |
| [Hosting, security, and observability](references/hosting-security-observability.md) | Metrics, exception diagnostics, OpenTelemetry, Kestrel, HTTP.sys, caching, compression, CSRF, certificates, SignalR |

## Breaking changes and upgrade checks

### Standalone Blazor WebAssembly environment

Do not use `Blazor-Environment`, `launchSettings.json`, or `ASPNETCORE_ENVIRONMENT` to select a standalone WebAssembly environment. Put the environment in the project:

```xml
<WasmApplicationEnvironmentName>Staging</WasmApplicationEnvironmentName>
```

Builds default to `Development`; publishes default to `Production`.

### Removed boot configuration and cache controls

`blazor.boot.json` is gone; its data is in `dotnet.js`. Workflows that edit or inspect the old file have no documented replacement. Remove `BlazorCacheBootResources`, because fingerprinted browser caching replaced the custom boot-resource cache.

### Blazor script inclusion

The compressed, fingerprinted Blazor script is included automatically only when the project has a `.razor` file. Force it into a component-free project with:

```xml
<RequiresAspNetWebAssets>true</RequiresAspNetWebAssets>
```

### WebAssembly response streams

Response streaming is on by default. `ReadAsStreamAsync` returns a `BrowserHttpReadStream`, which cannot perform synchronous reads. Opt out per request when a dependency requires a seekable or synchronously readable buffered stream:

```csharp
requestMessage.SetBrowserResponseStreamingEnabled(false);
```

For a global compatibility escape hatch, set `WasmEnableStreamingResponse` to `false` or `DOTNET_WASM_ENABLE_STREAMING_RESPONSE=0`.

### Router Not Found content

Replace the unsupported `<NotFound>` router fragment with `Router.NotFoundPage`. Use `NavigationManager.NotFound()` to set a 404 in static SSR or notify the interactive router, and subscribe to `NavigationManager.OnNotFound` for custom behavior.

### OpenAPI defaults and dependency migrations

Generated OpenAPI output now defaults to 3.2. Pin an earlier `OpenApiVersion` if a downstream consumer cannot parse 3.2.

Transformer code must also account for OpenAPI.NET's interface-based models, inline/reference implementations, `JsonSchemaType.Null` in place of `OpenApiSchema.Nullable`, and `JsonNode` in place of `OpenApiAny`. These API migrations apply even when output is pinned to 3.0.

### PipeReader JSON parsing

Custom `JsonConverter` implementations must not assume `Utf8JsonReader.ValueSpan` contains the whole token:

```csharp
var span = reader.HasValueSequence
    ? reader.ValueSequence.ToArray()
    : reader.ValueSpan;
```

The `Microsoft.AspNetCore.UseStreamBasedJsonParsing` AppContext switch is a temporary compatibility fallback.

### Handled-exception diagnostics

An exception handled by `IExceptionHandler` does not emit logs or other diagnostics by default. Restore reporting selectively with `ExceptionHandlerOptions.SuppressDiagnosticsCallback`; returning `false` means diagnostics are not suppressed.

### Kestrel request-target handling

Kestrel preserves `%2F` in HTTP/1.1 absolute-form paths. Middleware that expected `/a%2Fb` to become `/a/b` must treat the encoded segment explicitly.

Replace obsolete `TlsClientHelloBytesCallback` with `UseTlsClientHelloListener`, and register the listener before `UseHttps`.

### Automatic cross-origin form protection

Apps built with `WebApplication.CreateBuilder` automatically reject unsafe cross-origin browser form requests. Blazor Web Apps no longer need `UseAntiforgery()`. Use endpoint opt-outs only when intentional:

```csharp
app.MapPost("/trusted-hook", Handler)
   .DisableAntiforgery();
```

For application-wide policy, use `DisableCsrfProtection` or replace trust evaluation with `ICsrfProtection`.

### Preview API renames

Use the current API surface:

- `EnvironmentView`, not `EnvironmentBoundary`.
- `NavigationManager.GetUriWithFragment`, not `GetUriWithHash`.
- `WithBrowserOptions`, not `WithBrowserConfiguration`.
- `Ssr.PreserveDom`, with the opposite meaning from `DisableDomPreservation`.
- `CircuitInactivityTimeout` as a `TimeSpan`, not `CircuitInactivityTimeoutMs`.
- Type-, parameter-, and property-specific validation interfaces instead of `IValidatableInfo`.

### Testing top-level-statement apps

Remove a hand-written `public partial class Program`; the ASP.NET Core source generator emits the declaration needed by test projects.

## High-value feature patterns

### Recursive and asynchronous validation

Enable source-generated validation and mark root model types:

```csharp
builder.Services.AddValidation();

[ValidatableType]
public sealed class Order
{
    [Required]
    public string? Number { get; set; }
}
```

Keep models in C# files. Use `[SkipValidation]` for excluded graphs. Both the model assembly and app must call `AddValidation` when types cross assembly boundaries.

Static SSR forms with `DataAnnotationsValidator` validate client-side by default. Interactive validators can register tasks with `EditContext.AddValidationTask`, and callers can await `ValidateAsync`. Minimal APIs also run `AsyncValidationAttribute` and `IAsyncValidatableObject` after `AddValidation()`.

### Declarative state persistence

Use `[PersistentState]` for ordinary prerender persistence. Set `AllowUpdates = true` for enhanced-navigation refreshes, select a `RestoreBehavior` to skip an initial value or last snapshot, and register `PersistentComponentStateSerializer<T>` when JSON is not appropriate.

For SSR request-to-request values, use `[SupplyParameterFromTempData]` for consumable TempData or `[SupplyParameterFromSession]` for session-backed state. Configure the corresponding storage and middleware before relying on either.

### Navigation and virtualized UI

`NavigateTo` preserves same-page scroll position, and `NavLinkMatch.All` ignores query strings and fragments. Set `RelativeToCurrentUri` when a target should resolve against the current page rather than the base URI.

`Virtualize<TItem>` adapts to variable heights. Use `AnchorMode.End` for append-heavy feeds, `ItemComparer` for refreshed reference objects, `InitialIndex` for the starting position, and `ScrollToIndexAsync` only after the first interactive render.

### Circuit pause and browser configuration

Circuit state can survive long disconnects and graceful pauses, but not a full-page refresh. A server can call `RequestCircuitPauseAsync`; applications that need to target circuits must maintain their own registry through `CircuitHandler` callbacks.

Configure client startup from C#:

```csharp
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode()
    .WithBrowserOptions(options =>
    {
        options.Server.ReconnectionMaxRetries = 10;
        options.Ssr.PreserveDom = true;
    });
```

### Standalone WebAssembly development Gateway

Replace the WebAssembly DevServer package with `Microsoft.AspNetCore.Components.Gateway` when a full ASP.NET Core development host and SPA fallback are useful. Configure its built-in YARP `ReverseProxy` routes and clusters on the separate Gateway process through launch-profile environment variables or command-line arguments. Do not put them in the WebAssembly app's `appsettings.json`.

### OpenAPI response accuracy

Advertise concrete file results with `.Produces<T>(contentType: ...)` so generated schemas use `type: string`, `format: binary`. Multiple response declarations for one status are preserved by media type or `anyOf`, and non-body enum schemas intentionally use C# names to match binding.

OpenAPI 3.2 represents `QUERY` endpoints directly; 3.0 and 3.1 put them in `x-oai-additionalOperations`.

### Compression and cache correctness

Response compression and request decompression support zstd. Tune quality from 1 through 22, recognizing the speed/ratio tradeoff. Response compression emits `Vary: Accept-Encoding` even for uncompressed responses so shared caches do not mix representations.

### OpenTelemetry without duplicate instrumentation

Framework request activities contain required OpenTelemetry HTTP server attributes. Subscribe directly:

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddSource("Microsoft.AspNetCore")
        .AddConsoleExporter());
```

Do not add request instrumentation solely to obtain those attributes. Use `Microsoft.AspNetCore.Hosting.SuppressActivityOpenTelemetryData` only when suppression is intentional.

### SignalR authentication and cancellation

Set `EnableAuthenticationRefresh = true` on the hub endpoint so supported .NET clients refresh credentials before expiry without reconnecting. JavaScript clients and Azure SignalR do not yet support this flow.

Canceling the token passed to a regular .NET client `InvokeAsync` now propagates to the server hub method, not just to the client-side wait.
