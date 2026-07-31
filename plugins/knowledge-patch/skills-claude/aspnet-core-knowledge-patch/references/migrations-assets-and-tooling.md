# Migrations, Assets, and Tooling

Use this reference while upgrading an application or changing Blazor WebAssembly build output.
Treat project properties and generated assets as build contracts, and verify them in published
output.

## Standalone WebAssembly environment selection

Batch `10.0-migration` moves environment selection into the client project file:

```xml
<PropertyGroup>
  <WasmApplicationEnvironmentName>Staging</WasmApplicationEnvironmentName>
</PropertyGroup>
```

Standalone Blazor WebAssembly apps no longer select the environment from the
`Blazor-Environment` header, `Properties/launchSettings.json`, or `ASPNETCORE_ENVIRONMENT`.
Without the property, builds default to `Development` and publishes default to `Production`.
Check both local builds and published artifacts during migration.

## Boot configuration is part of `dotnet.js`

In batch `10.0-migration`, the contents formerly written to `blazor.boot.json` are inlined into
`dotnet.js`; the separate file no longer exists. Workflows that inspect or alter that file,
including the published-asset integrity script and DLL-extension customization, have no
documented replacement. Remove assumptions about the old file rather than silently fabricating
one.

## Remove the custom boot-resource cache property

Browser caching of fingerprinted client files replaces Blazor's custom boot-resource cache.
`BlazorCacheBootResources` is unavailable or ineffective and must be removed from upgraded
client project files (batch `10.0-migration`).

```diff
- <BlazorCacheBootResources>...</BlazorCacheBootResources>
```

Do not replace it with an equivalent service-worker customization unless the application has a
separate product requirement for one.

## Blazor script static-asset inclusion

The Blazor script is a compressed, fingerprinted static web asset in batch `10.0`. It is included
automatically only when the project contains at least one `.razor` file. A component-free project
that still needs the script must force inclusion:

```xml
<RequiresAspNetWebAssets>true</RequiresAspNetWebAssets>
```

Do not add the property routinely to component projects; automatic discovery already includes
the asset there.

## Standalone WebAssembly asset fingerprinting

To opt into build-time fingerprinting, enable HTML placeholder replacement, add an import map,
and put the fingerprint marker in the framework script name:

```xml
<PropertyGroup>
  <OverrideHtmlAssetPlaceholders>true</OverrideHtmlAssetPlaceholders>
</PropertyGroup>
```

```html
<script type="importmap"></script>
<script src="_framework/blazor.webassembly#[.{fingerprint}].js"></script>
```

Developer modules can use the same `#[.{fingerprint}]` marker through a
`StaticWebAssetFingerprintPattern`. Inspect the published HTML, import map, and filenames
together; a placeholder without its corresponding map and generated asset is incomplete.

## Bundler-friendly WebAssembly output

Published output can be shaped for JavaScript bundlers such as Webpack and Rollup:

```xml
<WasmBundlerFriendlyBootConfig>true</WasmBundlerFriendlyBootConfig>
```

This is an explicit publishing mode. Test the downstream bundler against published output and
retain any application-specific asset-copy rules only when they still match the new boot layout.

## Identity navigation migration

The Blazor Web App template uses:

```xml
<BlazorDisableThrowNavigationException>true</BlazorDisableThrowNavigationException>
```

When an older Individual Accounts app adopts this setting, update
`Components/Account/IdentityRedirectManager.cs` as described in
[Observability, Identity, and SignalR](observability-identity-and-signalr.md). Navigation helpers
must no longer claim that every redirect exits by throwing.

## Testing top-level-statement applications

The ASP.NET Core source generator emits the `public partial class Program` needed by test
projects in batch `10.0`. Remove an application's manual declaration when upgrading.

## Upgrade search checklist

Search for these migration-sensitive tokens and assumptions:

```text
Blazor-Environment
ASPNETCORE_ENVIRONMENT
blazor.boot.json
BlazorCacheBootResources
RequiresAspNetWebAssets
blazor.webassembly#[.{fingerprint}].js
WasmBundlerFriendlyBootConfig
BlazorDisableThrowNavigationException
partial class Program
```

Review each occurrence in context. Some tokens are required new configuration; others identify
behavior or assets that must be removed.
