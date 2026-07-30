# Blazor WebAssembly, Static Assets, and Tooling

## Standalone app migration and static assets

- Standalone Blazor WebAssembly environment selection moved into the project file. `Blazor-Environment`, `Properties/launchSettings.json`, and `ASPNETCORE_ENVIRONMENT` no longer select it. Builds default to `Development`; publishes default to `Production` (10.0-migration).

```xml
<WasmApplicationEnvironmentName>Staging</WasmApplicationEnvironmentName>
```

- `blazor.boot.json` no longer exists because boot configuration is inlined into `dotnet.js`. Workflows that inspect or mutate the old file—including the published-asset integrity script and DLL-extension customization—have no documented replacement (10.0-migration).
- Remove `BlazorCacheBootResources`; fingerprinted browser-cached files replaced Blazor's custom boot-resource cache, so the property is unavailable or ineffective (10.0-migration).
- The Blazor script is a compressed, fingerprinted static web asset and is automatically included only when the project contains a `.razor` file. A component-free app that still needs it must set `<RequiresAspNetWebAssets>true</RequiresAspNetWebAssets>` (10.0).
- Standalone apps can opt into build-time fingerprinting with `<OverrideHtmlAssetPlaceholders>true</OverrideHtmlAssetPlaceholders>`, an empty import map, and `_framework/blazor.webassembly#[.{fingerprint}].js`. Apply the same `#[.{fingerprint}]` marker to developer modules with a `StaticWebAssetFingerprintPattern` (10.0).
- Set `<WasmBundlerFriendlyBootConfig>true</WasmBundlerFriendlyBootConfig>` to produce published output suitable for JavaScript bundlers such as Webpack and Rollup (10.0).

## Browser runtime behavior

- WebAssembly `HttpClient` response streaming is enabled by default. `ReadAsStreamAsync` returns `BrowserHttpReadStream`, which does not support synchronous reads, instead of a `MemoryStream`. Disable streaming globally with `<WasmEnableStreamingResponse>false</WasmEnableStreamingResponse>` or `DOTNET_WASM_ENABLE_STREAMING_RESPONSE=0`, or per request with `requestMessage.SetBrowserResponseStreamingEnabled(false)` (10.0).
- `IJSRuntime` and `IJSObjectReference` support `InvokeConstructorAsync`, `GetValueAsync`, and `SetValueAsync` for direct JavaScript object construction and property access. In-process references have synchronous equivalents (10.0).
- Standalone apps load globalization resources for both `CultureInfo.DefaultThreadCurrentUICulture` and `DefaultThreadCurrentCulture` (10.0).
- WebAssembly starts and stops registered `IHostedService` implementations with the browser app lifecycle; register them with `builder.Services.AddHostedService<T>()` (11.0-preview.1).
- Environment variables are included automatically in WebAssembly configuration alongside sources such as `appsettings.json`, so values are available through `builder.Configuration` (11.0-preview.1).
- Prerendered Interactive WebAssembly persists the server's `CurrentCulture` and `CurrentUICulture` in component state and applies them before satellite assemblies load. Set `UseCultureFromServer = false` in `AddInteractiveWebAssemblyComponents` if the client must select culture independently (11.0-preview.5).

## Workers and generated projects

- The Web Worker template introduced as `webworker` provides `WebWorkerClient` for moving expensive WebAssembly work off the UI thread. Export methods with `[JSExport]`, create the worker with `WebWorkerClient.CreateAsync(JSRuntime)`, and call `InvokeAsync<T>` with the method's qualified name (11.0-preview.2).
- The template is now named `blazorwebworker`; projects generated under the earlier name remain valid. Its client adds `InvokeVoidAsync`, cancellation, and timeouts for creation and invocation (11.0-preview.4).

```bash
dotnet new blazorwebworker -o MyApp.Worker
```

- `blazor-wasm-servicedefaults` creates an Aspire-oriented client library with OpenTelemetry OTLP export, service discovery, and standard HTTP resilience. Reference it from the client and call `builder.AddBlazorClientServiceDefaults()` (11.0-preview.4).
- Visual Studio's Blazor Web App template offers **Enable container support** (11.0-preview.1).
- The .NET SDK includes the MCP server template; do not install `Microsoft.McpServer.ProjectTemplates` separately. Use `dotnet new mcpserver -o MyMcpServer` (11.0-preview.4).

## Standalone development Gateway

- Standalone projects can replace `Microsoft.AspNetCore.Components.WebAssembly.DevServer` with `Microsoft.AspNetCore.Components.Gateway`, a lightweight full ASP.NET Core development host. Its static-web-asset SPA fallback serves `index.html` for refreshes and direct client-route navigation without custom middleware (11.0-preview.5).

```xml
<PackageReference Include="Microsoft.AspNetCore.Components.Gateway"
                  Version="11.0.0-preview.5.26302.115"
                  PrivateAssets="all" />
```

- The Gateway includes a YARP reverse proxy so browser-to-backend calls can remain same-origin without client or backend CORS configuration. Give the separate Gateway process standard `ReverseProxy` routes and clusters through launch-profile environment variables or command-line arguments—not the app's `appsettings.json`. Cluster destinations may use .NET service-discovery names (11.0-preview.6).
