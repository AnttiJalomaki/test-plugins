# Hosting, platform behavior, and interop

## Hosting, configuration, and logging

.NET 10 changes hosting and extensions behavior (`10.0-guides`):

- The entire `BackgroundService.ExecuteAsync` invocation runs as a `Task`,
  including code before the first await.
- Configuration preserves null values.
- `ProviderAliasAttribute` moved to
  `Microsoft.Extensions.Logging.Abstractions`.
- Trim-related `DynamicallyAccessedMembers` annotations were removed from
  trim-unsafe `Microsoft.Extensions.Configuration` code.
- The ICU override variable is `DOTNET_ICU_VERSION_OVERRIDE`.

In .NET 11 Preview 6, `configProperties` values in
`.runtimeconfig.dev.json` override values from `.runtimeconfig.json`
(`11.0-preview.6-compatibility`). A failed `BackgroundService` propagates as an
exception from `IHost.RunAsync` and `IHost.StopAsync`.

## Native libraries and COM

Single-file .NET 10 applications do not probe the executable directory for
native libraries. `DllImportSearchPath.AssemblyDirectory` searches only the
assembly directory. Put native assets in an explicitly supported location or
configure loading directly.

Casting an `IDispatchEx` COM object to `IReflect` now fails. Do not use that
cast as a late-binding bridge.

## Runtime platform behavior

Linux `DriveInfo.DriveFormat` reports filesystem types in .NET 10.
LDAP `DirectoryControl` parsing is stricter.

## Windows desktop

.NET 10 desktop compatibility changes include:

- Projects referencing WPF and Windows Forms must disambiguate `MenuItem` and
  `ContextMenu`.
- `HtmlElement.InsertAdjacentElement` has a renamed parameter.
- `StatusStrip` defaults to the system render mode.
- Some `System.Drawing` failures throw `ExternalException` instead of
  `OutOfMemoryException`.
- WPF rejects empty `ColumnDefinitions` and `RowDefinitions`.
- Incorrect `DynamicResource` use can terminate the application.

Recompile source that uses renamed parameters, update exception handling, and
exercise XAML/resource startup paths during migration.

## MAUI and Android

.NET MAUI requires Android API level 24 or later in .NET 11 Preview 6. Update
the minimum target and remove devices below that level from the support
matrix.
