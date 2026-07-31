# Hosting, Platform Behavior, and Interop

## Hosting, configuration, and logging (`10.0-guides`)

### Background services

All of `BackgroundService.ExecuteAsync` now runs as a `Task`. Audit code that
depended on synchronous execution before the first await, especially startup
ordering and exception propagation.

### Configuration and logging

- Configuration preserves null values. Distinguish a present null from a
  missing key where configuration binding or merging depends on that state.
- `ProviderAliasAttribute` moved to
  `Microsoft.Extensions.Logging.Abstractions`; update package and namespace
  assumptions.
- `DynamicallyAccessedMembers` annotations were removed from trim-unsafe
  `Microsoft.Extensions.Configuration` code. Treat those paths as trim-unsafe
  and verify trimmed applications directly.
- The ICU override environment variable is `DOTNET_ICU_VERSION_OVERRIDE`.

## Process shutdown (`10.0-guides`)

The runtime no longer installs default termination-signal handlers. Applications
that require graceful termination must register the relevant handling and
connect it to their own shutdown lifecycle.

## Containers and native libraries (`10.0-guides`)

Default .NET 10 container images use Ubuntu. Builds that require packages,
paths, or a package manager from the prior distribution must pin a compatible
base or adapt the build.

Single-file applications no longer probe the executable directory for native
libraries. `DllImportSearchPath.AssemblyDirectory` searches only the assembly
directory. Package native dependencies in a searched location or configure
loading explicitly.

## COM interop (`10.0-guides`)

Casting an `IDispatchEx` COM object to `IReflect` now fails. Use supported COM
dispatch behavior instead of depending on that cast.

## Windows desktop compatibility (`10.0-guides`)

- A project referencing both WPF and Windows Forms must disambiguate
  `MenuItem` and `ContextMenu`.
- `HtmlElement.InsertAdjacentElement` has a renamed parameter. Update named
  arguments.
- `StatusStrip` defaults to the system render mode.
- Some `System.Drawing` failures throw `ExternalException` rather than
  `OutOfMemoryException`; revise exception handling that distinguishes them.
- WPF rejects empty `ColumnDefinitions` and `RowDefinitions`.
- Incorrect `DynamicResource` use can terminate the application. Validate
  resource keys and placement rather than relying on a recoverable failure.

## .NET tasks in .NET Framework MSBuild (`10.0`)

Visual Studio 2026 and `msbuild.exe` can execute .NET-built MSBuild tasks via
`TaskHostFactory`. The task runs out of process, and task Host Objects are not
supported on this path.

```xml
<UsingTask TaskName="MyTask"
           AssemblyFile="path\to\MyTask.dll"
           Runtime="NET"
           TaskFactory="TaskHostFactory" />
```

Add a conditional second `UsingTask` without the factory when Core MSBuild
should keep the task in process.
