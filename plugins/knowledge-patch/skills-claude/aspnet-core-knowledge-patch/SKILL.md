---
name: aspnet-core-knowledge-patch
description: ASP.NET Core
version: 10.0
license: MIT
metadata:
  author: Nevaberry
---



# ASP.NET Core Knowledge Patch

Use this skill when designing, upgrading, debugging, or reviewing ASP.NET Core applications.
Inspect the project file and target framework before applying version-sensitive guidance.
When the project targets a later release than the frontmatter version, prefer the project's
code, package APIs, generated output, and current documentation where they differ.

## How to use this skill

1. Identify the affected surface: Blazor components, forms and state, hosting and HTTP,
   migrations and assets, observability and identity, or OpenAPI and Minimal APIs.
2. Read the matching reference file before proposing code. Several changes alter defaults,
   generated assets, or compatibility behavior without producing a compiler error.
3. For upgrades, search the project for removed properties, old router fragments, custom JSON
   converters, OpenAPI transformer types, and a manually declared partial `Program` class.
4. Preserve explicit compatibility switches only as temporary migration aids. Prefer the new
   behavior in newly written code.
5. Validate browser-specific Blazor behavior in a published build when work involves
   fingerprinting, streaming, bundlers, globalization, reconnection, or static web assets.

## Reference index

| Reference | Topics |
| --- | --- |
| [Components, Navigation, and Interop](references/components-navigation-and-interop.md) | Navigation, routing, reconnection, circuits, JavaScript interop, globalization |
| [Forms, Validation, and Persistence](references/forms-validation-and-persistence.md) | Recursive validation, form binding, prerendered state, serializers, restoration |
| [Hosting, HTTP, Caching, and Security](references/hosting-http-caching-and-security.md) | Browser HTTP streaming, exception diagnostics, Kestrel, JSON parsing, memory pools, HTTP.sys |
| [Migrations, Assets, and Tooling](references/migrations-assets-and-tooling.md) | Upgrade removals, Blazor environment selection, boot assets, fingerprinting, bundlers, testing |
| [Observability, Identity, and SignalR](references/observability-identity-and-signalr.md) | Authentication and Identity metrics, passkey migration, Identity redirects |
| [OpenAPI, Minimal APIs, and JSON](references/openapi-minimal-apis-and-json.md) | OpenAPI 3.1, OpenAPI.NET 2, XML comments, transformer schemas, form binding |

## Breaking changes and migration traps

### Configure standalone WebAssembly environments in the project

Do not use the `Blazor-Environment` response header, `launchSettings.json`, or
`ASPNETCORE_ENVIRONMENT` to select a standalone Blazor WebAssembly environment. Set:

```xml
<WasmApplicationEnvironmentName>Staging</WasmApplicationEnvironmentName>
```

Builds otherwise use `Development`; publishes use `Production`.

### Stop depending on `blazor.boot.json`

Boot configuration is inlined into `dotnet.js`; the separate `blazor.boot.json` asset no
longer exists. Do not port workflows that inspect or mutate it unless a supported replacement
is available. Remove `BlazorCacheBootResources`; fingerprinted browser assets now provide the
cache behavior.

### Force the Blazor script only for component-free projects

The compressed, fingerprinted Blazor script is automatically included when the project has a
`.razor` file. A project that needs the script but has no component must opt in:

```xml
<RequiresAspNetWebAssets>true</RequiresAspNetWebAssets>
```

### Account for response streaming in WebAssembly

`ReadAsStreamAsync` returns a `BrowserHttpReadStream` by default and synchronous reads fail.
Keep consumers asynchronous, or opt out for a specific request:

```csharp
requestMessage.SetBrowserResponseStreamingEnabled(false);
```

Global compatibility controls are `WasmEnableStreamingResponse=false` and
`DOTNET_WASM_ENABLE_STREAMING_RESPONSE=0`.

### Update routing assumptions

- Same-page `NavigateTo` calls preserve scroll position when only the query or fragment changes.
- `NavLinkMatch.All` compares the path and ignores query strings and fragments.
- Use `NavigationManager.NotFound()`, `Router.NotFoundPage`, and `OnNotFound`; the router's old
  `<NotFound>` fragment is unsupported.

Temporarily restore query-and-fragment matching with the
`Microsoft.AspNetCore.Components.Routing.NavLink.EnableMatchAllForQueryStringAndFragment`
AppContext switch.

### Migrate OpenAPI transformers

OpenAPI entities are interfaces with separate inline and reference implementations.
Replace `OpenApiSchema.Nullable` checks with `JsonSchemaType.Null` checks and replace
`OpenApiAny` with `JsonNode`. These API migrations are required even when emitting an
OpenAPI 3.0 document.

### Make custom JSON converters sequence-safe

ASP.NET Core deserializes MVC, Minimal API, and `ReadFromJsonAsync` payloads through
`PipeReader`. A converter must handle segmented values:

```csharp
var span = reader.HasValueSequence
    ? reader.ValueSequence.ToArray()
    : reader.ValueSpan;
```

The temporary `Microsoft.AspNetCore.UseStreamBasedJsonParsing` AppContext switch restores
stream-based parsing.

### Remove a manual test-visible `Program` declaration

Top-level-statement web apps now receive a generated `public partial class Program` for test
projects. Remove the manual declaration to avoid duplication.

## High-value capabilities

### Enable recursive Blazor validation

Register validation, keep model types in C# files, and mark the root model:

```csharp
builder.Services.AddValidation();

[ValidatableType]
public sealed class Order
{
    [Required]
    public string? Number { get; set; }
}
```

Nested objects and collections are then validated without reflection. Use `[SkipValidation]`
for exclusions; validation across assemblies requires registration in both assemblies.

### Use direct JavaScript object interop

Construct JavaScript objects and access their properties through `IJSRuntime` and
`IJSObjectReference`:

```csharp
var instance = await JSRuntime.InvokeConstructorAsync("jsInterop.TestClass", "Blazor!");
var text = await instance.GetValueAsync<string>("text");
await instance.SetValueAsync("text", "updated");
```

In-process references provide synchronous equivalents.

### Persist and resume component state deliberately

Use `[PersistentState]` for declarative prerender persistence. Set `AllowUpdates = true` for
enhanced-navigation updates, select `RestoreBehavior.SkipInitialValue` or `SkipLastSnapshot`
when restoration is inappropriate, and register `PersistentComponentStateSerializer<T>` for
custom serialization. Server circuits can resume after extended disconnects or proactive pauses
as long as the browser does not perform a full-page refresh.

### Generate richer OpenAPI documents

Generated documents default to OpenAPI 3.1. Enable `GenerateDocumentationFile` to flow XML
summaries, remarks, parameter descriptions, returns, and referenced-project comments into the
document. Use documented methods rather than Minimal API lambdas when XML metadata matters.
Transformer contexts can call `GetOrCreateSchemaAsync`; operation and schema contexts expose
the document so generated schemas can be registered with `AddComponent`.

### Control handled-exception diagnostics

Exceptions handled by `IExceptionHandler` suppress logs and other diagnostics by default.
Restore reporting globally, or choose exceptions selectively, with:

```csharp
app.UseExceptionHandler(new ExceptionHandlerOptions
{
    SuppressDiagnosticsCallback = context => false
});
```

### Observe authentication and Identity

Use the built-in authentication duration and event counters for challenge, forbid, sign-in,
sign-out, and authorization. Identity emits through the `Microsoft.AspNetCore.Identity` meter,
including user-creation, password-check, and sign-in instruments.

## Review checklist

- Confirm standalone WebAssembly environment and asset properties in the project file.
- Test published static assets and import maps, not only development output.
- Keep browser stream consumers asynchronous.
- Exercise query, fragment, not-found, reconnect, and full-refresh navigation paths.
- Validate nested models and nullable form values with realistic submissions.
- Inspect generated OpenAPI schemas under the configured JSON number handling.
- Update custom converters for multi-segment `Utf8JsonReader` input.
- Decide explicitly whether handled exceptions should emit diagnostics.
- Re-trust the development certificate after adopting a `*.dev.localhost` template domain.
- Verify HTTP.sys queue ACL changes are applied only when creating a new request queue.
