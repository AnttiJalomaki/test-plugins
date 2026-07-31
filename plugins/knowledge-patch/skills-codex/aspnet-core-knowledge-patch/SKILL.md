---
name: aspnet-core-knowledge-patch
description: ASP.NET Core
version: 10.0
license: MIT
metadata:
  author: Nevaberry
---


# ASP.NET Core Compatibility Guidance

Use this skill when updating or reviewing ASP.NET Core applications that use
Blazor, Minimal APIs, MVC, OpenAPI generation, Kestrel, HTTP.sys, Identity, or
the ASP.NET Core testing infrastructure.

## Start Here

Before changing code:

1. Inspect the target framework and relevant project properties.
2. Identify whether the application uses Blazor WebAssembly, static SSR,
   interactive server rendering, generated OpenAPI, or custom JSON converters.
3. Treat project files, package versions, generated output, and tests as the
   source of truth when they differ from a migration assumption.
4. Read the focused reference for the subsystem being changed.

## Reference Index

| Reference | Topics |
| --- | --- |
| [Blazor Components, Navigation, and Circuits](references/blazor-components-navigation.md) | Navigation, Not Found handling, reconnection, circuit state, and JavaScript objects |
| [Blazor WebAssembly, Static Assets, and Tooling](references/blazor-wasm-tooling.md) | Environments, boot configuration, caching, fingerprinting, streaming, bundlers, and globalization |
| [Forms, Validation, Persistent State, and Identity](references/forms-validation-state-identity.md) | Recursive validation, form binding, persistent state, passkeys, and Identity redirects |
| [Hosting, Security, Compression, and Observability](references/hosting-security-observability.md) | Diagnostics, metrics, development domains, memory pools, HTTP.sys, and testing |
| [OpenAPI, HTTP APIs, and JSON](references/openapi-http-apis.md) | OpenAPI 3.1, transformer migration, XML documentation, schema generation, and `PipeReader` JSON |

## Breaking Changes and Migration Priorities

### Move standalone WebAssembly environment selection into the project

Do not use `Blazor-Environment`, `launchSettings.json`, or
`ASPNETCORE_ENVIRONMENT` to select a standalone Blazor WebAssembly
environment. Set the project property explicitly when the defaults are not
appropriate:

```xml
<WasmApplicationEnvironmentName>Staging</WasmApplicationEnvironmentName>
```

Builds otherwise use `Development`; published apps use `Production`.

### Remove obsolete boot-file and cache customization

The boot configuration is inlined into `dotnet.js`; `blazor.boot.json` no
longer exists. Do not design an upgrade around editing or inspecting that
file. There is no documented replacement for the old integrity and DLL-file
extension customization workflows.

Remove `BlazorCacheBootResources` from client project files. Fingerprinted
browser assets now provide the caching behavior.

### Update OpenAPI transformers for OpenAPI.NET 2

Transformer code must account for interface-based OpenAPI entities and their
separate inline and reference implementations. Replace:

- `OpenApiSchema.Nullable` checks with `JsonSchemaType.Null` checks.
- `OpenApiAny` values with `JsonNode`.

This migration is necessary even if document generation is configured for
OpenAPI 3.0.

### Make custom JSON converters sequence-aware

ASP.NET Core JSON input now arrives through `PipeReader`. A converter must not
assume `Utf8JsonReader.ValueSpan` contains the complete token:

```csharp
var span = reader.HasValueSequence
    ? reader.ValueSequence.ToArray()
    : reader.ValueSpan;
```

Use the `Microsoft.AspNetCore.UseStreamBasedJsonParsing` AppContext switch only
as a temporary compatibility measure.

### Update Blazor Not Found pages

The router's old `<NotFound>` fragment is unsupported. Set
`Router.NotFoundPage`, call `NavigationManager.NotFound()` when application
logic discovers a missing resource, and use `NavigationManager.OnNotFound`
when custom handling is required.

```razor
<Router AppAssembly="@typeof(Program).Assembly"
        NotFoundPage="typeof(Pages.NotFound)">
    <Found Context="routeData">
        <RouteView RouteData="@routeData" />
    </Found>
</Router>
```

### Expect streamed WebAssembly HTTP responses

Blazor WebAssembly response streaming is enabled by default.
`ReadAsStreamAsync` returns `BrowserHttpReadStream`, which does not support
synchronous reads. Disable streaming only where compatibility requires it:

```csharp
requestMessage.SetBrowserResponseStreamingEnabled(false);
```

For a global opt-out, set `WasmEnableStreamingResponse` to `false` or
`DOTNET_WASM_ENABLE_STREAMING_RESPONSE=0`.

## High-Value Blazor Behavior

### Include the Blazor script when no component exists

The compressed, fingerprinted Blazor script is included automatically only
when the project contains a `.razor` file. Force inclusion for component-free
projects that still need the script:

```xml
<RequiresAspNetWebAssets>true</RequiresAspNetWebAssets>
```

### Account for navigation matching changes

Same-page `NavigateTo` calls preserve scroll position when only the query or
fragment changes. `NavLinkMatch.All` now matches only the path, ignoring query
and fragment values. Restore its earlier matching behavior only when needed:

```csharp
AppContext.SetSwitch(
    "Microsoft.AspNetCore.Components.Routing.NavLink.EnableMatchAllForQueryStringAndFragment",
    true);
```

### Use the reconnection event and state model

The template reconnection UI collocates its CSS and JavaScript, making it
compatible with strict CSP `style-src` policies. Listen for
`components-reconnect-state-changed`; handle the `retrying` state in addition
to the other reconnection states.

### Use direct JavaScript object APIs

`IJSRuntime` and `IJSObjectReference` can construct JavaScript objects and get
or set their properties:

```csharp
var instance = await JSRuntime.InvokeConstructorAsync(
    "jsInterop.TestClass", "Blazor!");
var text = await instance.GetValueAsync<string>("text");
await instance.SetValueAsync("text", "updated");
```

In-process references provide synchronous equivalents.

## State, Validation, and Identity

### Prefer declarative persistent state

Use `[PersistentState]` on component or service state instead of manually
coordinating the common `PersistentComponentState` pattern. For advanced
behavior, use `AllowUpdates`, `RestoreBehavior`, custom
`PersistentComponentStateSerializer<T>` registrations, or
`RegisterOnRestoring`.

### Enable recursive source-generated validation

Call `AddValidation`, place the model in a C# file, and annotate its root with
`[ValidatableType]`. Use `[SkipValidation]` for excluded properties or types.
Both the application and an external model assembly must call `AddValidation`
when validation metadata crosses assembly boundaries.

```csharp
builder.Services.AddValidation();

[ValidatableType]
public sealed class Order
{
    [Required]
    public string? Number { get; set; }
}
```

### Fix upgraded Individual Accounts redirects

When enabling `BlazorDisableThrowNavigationException` in an older Blazor Web
App, remove the `InvalidOperationException` thrown by `RedirectTo` and all five
`[DoesNotReturn]` attributes from
`Components/Account/IdentityRedirectManager.cs`.

## OpenAPI and Hosting Defaults

### Review generated OpenAPI schemas

Generated documents default to OpenAPI 3.1. Nullable scalar schemas use a type
array containing `null`; nullable complex types and collections use `oneOf`.
Because the default JSON number handling permits reading numbers from strings,
`int` and `long` may be represented by digit patterns without
`type: integer`. Configure strict number handling when integer schemas are
required.

### Restore diagnostics deliberately

Exceptions handled by `IExceptionHandler` no longer emit diagnostics by
default. Report selected handled exceptions—or restore the earlier behavior—
with `ExceptionHandlerOptions.SuppressDiagnosticsCallback`:

```csharp
app.UseExceptionHandler(new ExceptionHandlerOptions
{
    SuppressDiagnosticsCallback = context => false
});
```

### Re-trust the development certificate for custom localhost domains

Kestrel treats `*.localhost` as loopback bindings. The `web` and `blazor`
templates accept `--localhost-tld`; after choosing a domain such as
`<project>.dev.localhost`, trust the development certificate again:

```console
dotnet dev-certs https --trust
```

## Review Checklist

- Remove obsolete WebAssembly boot-file and cache assumptions.
- Verify streamed response consumers do not perform synchronous reads.
- Update router Not Found markup and `NavLinkMatch.All` expectations.
- Check OpenAPI transformers and nullable/integer schema assertions.
- Make custom JSON converters handle multi-segment values.
- Verify handled exceptions still produce any diagnostics operations require.
- Use the topic references for less common hosting, Identity, state, and
  testing details before completing an upgrade.
