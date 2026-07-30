# SDK, CLI, packaging, testing, and containers

## CLI output and command defaults

.NET 10 CLI behavior changes (`10.0-guides`):

- `--interactive` defaults to `true` in user scenarios.
- Output unrelated to a command's result goes to stderr.
- `dotnet watch` logs to stderr.
- `dotnet new sln` creates SLNX by default.
- `dotnet package list` performs a restore.
- `dotnet tool install --local` creates a tool manifest if one is absent.

Automation should parse result data from stdout and capture stderr separately
for progress and diagnostics.

## Command discovery and completions

Every CLI command accepts `--cli-schema` to emit a JSON description of its
arguments, options, and subcommands (`10.0`):

```bash
dotnet clean --cli-schema
```

Noun-first aliases `dotnet package add|list|remove` and
`dotnet reference add|list|remove` coexist with the older verb-first commands.
`dotnet completions script` generates native scripts for Bash, Fish, Nushell,
PowerShell, and Zsh.

```bash
dotnet completions script bash
```

## One-shot and platform-specific tools

`dotnet tool exec` downloads and runs a tool without installing it. It chooses
the latest version unless the package is written as `package@version`, asks
before a new download, and honors the version in a nearby local tool manifest.

```bash
dotnet tool exec \
  --source ./artifacts/package \
  dotnetsay@0.1.0 "Hello"
```

Tool packaging creates runtime-identifier-specific packages. Include the `any`
RID alongside platform RIDs to add a framework-dependent, platform-neutral
fallback for machines without a more specific binary:

```xml
<RuntimeIdentifiers>linux-x64;win-x64;any</RuntimeIdentifiers>
```

## File-based applications

Publishing a file-based app with `dotnet publish app.cs` creates a native
executable by default. Set `PublishAot=false` for dependencies incompatible
with native AOT. File-based apps accept `#:project` and extensionless
executable files with a shebang.

```csharp
#!/usr/bin/env dotnet
#:project ../ClassLib/ClassLib.csproj
#:property PublishAot=false
Console.WriteLine(new ClassLib.Greeter().Greet());
```

.NET 11 Preview 6 adds `#:include` for composing an app from source files or
prebuilt DLL references (`11.0-preview.6`). Matching duplicate `#:sdk`,
`#:property`, and `#:package` directives are allowed across included files.

```csharp
#:include helpers.cs
#:include ./libs/MyLibrary.dll
```

The .NET 10 SDK rejects double quotes in file-level directives. `dnx` scripts
bypass `global.json` SDK selection, and `dnx.ps1` is no longer shipped.

## Run and watch workflows

`dotnet run -e KEY=VALUE` passes environment variables to the application and
exposes them to MSBuild as `RuntimeEnvironmentVariable` items.

```bash
dotnet run -e ASPNETCORE_ENVIRONMENT=Development
```

`dotnet watch` integrates with Aspire, relaunches a crashed app after the next
relevant edit, and selects mobile devices with `--device`.

```bash
dotnet watch --device device-id
```

## Solutions and MSBuild tasks

`dotnet sln` can create and edit `.slnf` solution filters:

```bash
dotnet new slnf --name MyApp.slnf
dotnet sln MyApp.slnf add src/Lib/Lib.csproj
```

Visual Studio 2026 and `msbuild.exe` can run .NET-built MSBuild tasks out of
process with `TaskHostFactory`:

```xml
<UsingTask TaskName="MyTask"
           AssemblyFile="path\to\MyTask.dll"
           Runtime="NET"
           TaskFactory="TaskHostFactory" />
```

This host does not support task Host Objects. A conditional second
`UsingTask`, without the factory, can retain in-process execution under Core
MSBuild.

Target-framework `DefineConstants` are not available during evaluation in the
.NET 10 SDK.

## Test runners

`dotnet test` can use Microsoft.Testing.Platform when selected in
`global.json` (`10.0`):

```json
{
  "test": {
    "runner": "Microsoft.Testing.Platform"
  }
}
```

In .NET 11 Preview 6, `DOTNET_TEST_RUNNER` selects `VSTest` or
`Microsoft.Testing.Platform` without changing `global.json`. MTP adds:

- `--no-dependencies`.
- `--use-current-runtime`.
- Exclusion patterns in `--test-modules`.
- Two-stage Ctrl+C cancellation.
- Live output.
- MAUI device selection through `--device`.

The xUnit template can create a v3 project that defaults to MTP, while NUnit
can opt into MTP explicitly:

```bash
dotnet new xunit --xunit-version v3
dotnet new nunit --test-runner Microsoft.Testing.Platform
```

VSTest no longer brings `Newtonsoft.Json` transitively in .NET 11 Preview 6
(`11.0-preview.6-compatibility`). Integrations that use it must declare their
own compatible dependency.

## Container publishing

The default .NET 10 container image distribution is Ubuntu. Rework or pin any
build that assumes packages, paths, or the package manager of the prior base.

Console projects can run:

```bash
dotnet publish /t:PublishContainer
```

They no longer need `EnableSdkContainerSupport`. Set `ContainerImageFormat` to
`Docker` or `OCI` instead of inheriting a default that can vary with the base
image and multi-architecture mode:

```xml
<PropertyGroup>
  <ContainerImageFormat>OCI</ContainerImageFormat>
</PropertyGroup>
```

The .NET 11 SDK publisher can create multi-architecture images with Podman.

## Workloads, SDK environment, and coverage

Workload management defaults to workload-set mode rather than loose manifests
in .NET 10. Dynamic native code-coverage instrumentation defaults to false.

When `DOTNET_CLI_USE_MSBUILD_SERVER` is unset in .NET 11 Preview 6, the CLI no
longer forces `MSBUILDUSESERVER=0`. Any standard `OTEL_EXPORTER_OTLP_*`
variable enables the CLI OTLP exporter as well as the dedicated exporter
flag.

## NuGet restore, audit, and pruning

.NET 10 changes package operations as follows:

- `dotnet restore` audits transitive packages.
- A versionless `PackageReference` is an error.
- Direct references pruned by NuGet produce NU1510.
- `PrunePackageReference` makes direct prunable references private.
- Packages with no runtime assets are omitted from `deps.json`.
- HTTP warnings in package list or search are errors.
- Invalid package IDs are errors.
- SHA-1 signing fingerprints are deprecated.
- `NUGET_ENABLE_ENHANCED_HTTP_RETRY` is removed.

In .NET 11 Preview 6, NuGet emits NU1703 for fallback to deprecated
MonoAndroid framework assets. `NuGet pack` emits NU5052 when a package ID
contains restricted characters.

## Target and template compatibility

The SDK no longer sets the Mono launch target for .NET Framework applications
in .NET 11 Preview 6. Workflows that depended on the inferred target must set
their launch behavior explicitly.

.NET template-engine packages no longer support `netstandard2.0`, which is a
source and binary compatibility break for consumers on that target.
