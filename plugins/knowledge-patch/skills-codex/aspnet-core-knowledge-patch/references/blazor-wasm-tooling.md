# Blazor WebAssembly, Static Assets, and Tooling

## Set standalone environments in the project

For the `10.0-migration`, standalone Blazor WebAssembly apps no longer use the
`Blazor-Environment` header, `Properties/launchSettings.json`, or
`ASPNETCORE_ENVIRONMENT` to select the environment. Builds default to
`Development`, while publishes default to `Production`. Override the value in
the project file:

```xml
<WasmApplicationEnvironmentName>Staging</WasmApplicationEnvironmentName>
```

## Stop reading or rewriting `blazor.boot.json`

The boot configuration is now inlined into `dotnet.js`, so the separate
`blazor.boot.json` file no longer exists. Workflows that inspected or modified
that file must be removed. No replacement is documented for either the
published-asset integrity script or DLL-extension customization.

## Remove the custom boot-resource cache setting

Browser caching of fingerprinted client files replaces Blazor's custom cache.
`BlazorCacheBootResources` is unavailable or ineffective and must be removed
from client project files:

```diff
- <BlazorCacheBootResources>...</BlazorCacheBootResources>
```

## Force Blazor script inclusion when necessary

In `10.0`, the Blazor script is a compressed, fingerprinted static web asset.
It is automatically included only if the project has a `.razor` file. A
component-free project that still uses the script must opt in:

```xml
<RequiresAspNetWebAssets>true</RequiresAspNetWebAssets>
```

## Handle streamed HTTP responses

Response streaming is enabled by default. `ReadAsStreamAsync` returns a
`BrowserHttpReadStream` instead of a `MemoryStream`, and the browser stream
does not support synchronous reads.

Disable streaming for one request:

```csharp
requestMessage.SetBrowserResponseStreamingEnabled(false);
```

Use `<WasmEnableStreamingResponse>false</WasmEnableStreamingResponse>` or
`DOTNET_WASM_ENABLE_STREAMING_RESPONSE=0` for a global opt-out.

## Fingerprint standalone assets

Standalone apps can opt into build-time fingerprinting by enabling HTML asset
placeholder replacement, adding an import map, and putting the fingerprint
marker in the framework script filename:

```xml
<OverrideHtmlAssetPlaceholders>true</OverrideHtmlAssetPlaceholders>
```

```html
<script type="importmap"></script>
<script src="_framework/blazor.webassembly#[.{fingerprint}].js"></script>
```

Developer modules can use the same `#[.{fingerprint}]` marker through a
`StaticWebAssetFingerprintPattern` item.

## Produce bundler-friendly output

Published output can be made compatible with JavaScript bundlers such as
Webpack and Rollup:

```xml
<WasmBundlerFriendlyBootConfig>true</WasmBundlerFriendlyBootConfig>
```

## Load UI-culture resources

Standalone apps load globalization resources for
`CultureInfo.DefaultThreadCurrentUICulture` as well as those selected by
`DefaultThreadCurrentCulture`. Account for the UI culture when auditing
downloaded globalization data.
