# Migrations, Assets, and Tooling

## Contents

- [Standalone WebAssembly migration](#standalone-webassembly-migration)
- [Blazor script and asset fingerprinting](#blazor-script-and-asset-fingerprinting)
- [Bundler-friendly output](#bundler-friendly-output)
- [WebAssembly Gateway](#webassembly-gateway)
- [Project and service templates](#project-and-service-templates)
- [Web Workers](#web-workers)
- [Testing top-level-statement applications](#testing-top-level-statement-applications)

## Standalone WebAssembly migration

### Select the environment in the project file

Standalone Blazor WebAssembly no longer uses the `Blazor-Environment` header,
`Properties/launchSettings.json`, or `ASPNETCORE_ENVIRONMENT` to choose its
environment (10.0-migration). Set it in the project:

```xml
<WasmApplicationEnvironmentName>Staging</WasmApplicationEnvironmentName>
```

Without an explicit value, builds default to `Development` and publishes
default to `Production`.

### Stop reading `blazor.boot.json`

The separate `blazor.boot.json` file no longer exists; its content is inlined
into `dotnet.js` (10.0-migration). Workflows that inspect or alter the old
file—including published-asset integrity scripts and DLL-extension
customization—have no documented replacement.

### Remove the custom boot cache property

Browser caching of fingerprinted client assets replaces Blazor's custom
boot-resource caching (10.0-migration). Remove
`BlazorCacheBootResources`; it is unavailable or ineffective.

```diff
- <BlazorCacheBootResources>...</BlazorCacheBootResources>
```

## Blazor script and asset fingerprinting

### Force script inclusion when there are no components

The Blazor script is a compressed, fingerprinted static web asset. It is
included automatically only when a project contains a `.razor` file
(10.0). A project that needs the script without a component must opt in:

```xml
<RequiresAspNetWebAssets>true</RequiresAspNetWebAssets>
```

### Fingerprint standalone assets

Standalone WebAssembly apps can opt into build-time fingerprinting by enabling
HTML placeholder replacement, adding an import map, and placing the
fingerprint marker in the framework script name (10.0).

```xml
<OverrideHtmlAssetPlaceholders>true</OverrideHtmlAssetPlaceholders>
```

```html
<script type="importmap"></script>
<script src="_framework/blazor.webassembly#[.{fingerprint}].js"></script>
```

Developer modules can use the same `#[.{fingerprint}]` marker through a
`StaticWebAssetFingerprintPattern`.

## Bundler-friendly output

Set `WasmBundlerFriendlyBootConfig` when published WebAssembly output must be
consumed by a JavaScript bundler such as Webpack or Rollup (10.0):

```xml
<WasmBundlerFriendlyBootConfig>true</WasmBundlerFriendlyBootConfig>
```

## WebAssembly Gateway

### Replace the development server

Standalone WebAssembly projects can replace
`Microsoft.AspNetCore.Components.WebAssembly.DevServer` with
`Microsoft.AspNetCore.Components.Gateway`, a lightweight full ASP.NET Core
development host (11.0-preview.5).

```xml
<PackageReference Include="Microsoft.AspNetCore.Components.Gateway"
                  Version="11.0.0-preview.5.26302.115"
                  PrivateAssets="all" />
```

The Gateway setup uses static-web-asset SPA fallback endpoints. Refreshing or
directly opening a client route serves `index.html` without custom fallback
middleware.

### Proxy the backend

The Gateway can use its built-in YARP reverse proxy so browser-to-backend
calls stay same-origin and do not require client or backend CORS configuration
(11.0-preview.6).

Configure standard `ReverseProxy` routes and clusters for the separate Gateway
process through launch-profile environment variables or command-line
arguments. Do not put those settings in the application's `appsettings.json`.
Cluster destinations may use .NET service-discovery names.

## Project and service templates

### Container-enabled Blazor template

The Blazor Web App project template in Visual Studio includes an
**Enable container support** option (11.0-preview.1).

### WebAssembly service defaults

The `blazor-wasm-servicedefaults` template creates an Aspire-oriented client
library that configures OpenTelemetry with OTLP export, service discovery, and
standard HTTP resilience (11.0-preview.4). Reference it from the WebAssembly
client and call its generated startup extension.

```bash
dotnet new blazor-wasm-servicedefaults -o MyApp.ServiceDefaults
```

```csharp
builder.AddBlazorClientServiceDefaults();
```

### MCP server template

The .NET SDK bundles the MCP server template, so a separate
`Microsoft.McpServer.ProjectTemplates` installation is not required
(11.0-preview.4).

```bash
dotnet new mcpserver -o MyMcpServer
```

## Web Workers

The Web Worker template supplies a `WebWorkerClient` for moving expensive
WebAssembly work off the UI thread. Export worker methods with `[JSExport]`,
then create and invoke the worker using the method's qualified name
(11.0-preview.2).

The original template name `webworker` was renamed to `blazorwebworker` in
11.0-preview.4. Existing projects generated with the old name remain valid.
The later client also supports `InvokeVoidAsync`, cancellation, and timeouts
for worker creation and invocation.

```bash
dotnet new blazorwebworker -o MyApp.Worker
```

```csharp
await using var worker = await WebWorkerClient.CreateAsync(JSRuntime);
var result = await worker.InvokeAsync<string>(
    "MyApp.MyWorker.Greet", ["World"]);
```

## Testing top-level-statement applications

The ASP.NET Core source generator emits the `public partial class Program`
needed by test projects (10.0). Remove a manually declared partial `Program`
from applications that use top-level statements.
