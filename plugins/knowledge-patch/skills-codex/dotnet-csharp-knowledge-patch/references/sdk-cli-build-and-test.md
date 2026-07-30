# SDK, CLI, Build, Restore, and Test

Use this reference for CLI contracts, tools, file-based applications, MSBuild,
SDK evaluation, workloads, NuGet, testing, solution filters, and run/watch
workflows. It consolidates `10.0-guides`, `10.0`,
`11.0-preview.6-compatibility`, and `11.0-preview.6`.

## CLI defaults and automation

The following defaults changed in `10.0-guides`:

- `--interactive` defaults to `true` in user scenarios.
- Output not directly relevant to the command goes to standard error.
- `dotnet watch` logs to standard error.
- `dotnet new sln` creates SLNX by default.
- `dotnet package list` performs a restore.
- `dotnet tool install --local` creates a tool manifest if one is absent.

Automation should capture stdout and stderr separately and should explicitly
select noninteractive behavior where prompts are unacceptable. Scripts that
expect `.sln` should request or preserve that format rather than relying on
the new template default.

Every CLI command accepts `--cli-schema` to emit a JSON description of its
arguments, options, and subcommands:

```bash
dotnet clean --cli-schema
```

Use the schema for shell integration and tooling rather than scraping help
text.

Noun-first aliases coexist with older verb-first forms:

```text
dotnet package add|list|remove
dotnet reference add|list|remove
```

`dotnet completions script` emits native completion scripts for Bash, Fish,
Nushell, PowerShell, and Zsh:

```bash
dotnet completions script bash
```

## Tool execution and packaging

Run a package as a one-shot tool without installing it:

```bash
dotnet tool exec \
  --source ./artifacts/package \
  dotnetsay@0.1.0 "Hello"
```

`dotnet tool exec` uses the latest version unless `package@version` is
specified, prompts before a new download, and honors the version in a nearby
local tool manifest (`10.0`).

.NET tool packaging creates runtime-identifier-specific packages
(`10.0-guides`). Include the `any` RID with platform RIDs to provide a
framework-dependent, platform-agnostic fallback:

```xml
<RuntimeIdentifiers>linux-x64;win-x64;any</RuntimeIdentifiers>
```

## File-based applications

Publishing a file-based app with `dotnet publish app.cs` produces a native
executable because native AOT is the default. Disable it for incompatible
dependencies:

```csharp
#!/usr/bin/env dotnet
#:project ../ClassLib/ClassLib.csproj
#:property PublishAot=false
Console.WriteLine(new ClassLib.Greeter().Greet());
```

File-based apps accept `#:project` references and executable extensionless
shebang files (`10.0`).

In `11.0-preview.6`, `#:include` composes an app from source files or prebuilt
DLL references:

```csharp
#:include helpers.cs
#:include ./libs/MyLibrary.dll
```

Matching duplicate `#:sdk`, `#:property`, and `#:package` directives are
allowed across included files.

Double quotes in file-level directives are rejected (`10.0-guides`). Use the
directive's supported unquoted syntax.

`dnx` scripts bypass `global.json` SDK selection, and `dnx.ps1` is no longer
included. Do not assume script execution uses the repository-pinned SDK or
that the PowerShell shim exists.

## Solution filters

`dotnet sln` can create and edit `.slnf` solution filters
(`11.0-preview.6`):

```bash
dotnet new slnf --name MyApp.slnf
dotnet sln MyApp.slnf add src/Lib/Lib.csproj
```

Use solution filters for a CLI-managed subset of the main solution.

## Run and watch

`dotnet run -e KEY=VALUE` passes environment variables to the launched
application and exposes them to MSBuild as `RuntimeEnvironmentVariable`
items:

```bash
dotnet run -e ASPNETCORE_ENVIRONMENT=Development
```

`dotnet watch` integrates with Aspire, relaunches a crashed application after
the next relevant edit, and supports mobile device selection:

```bash
dotnet watch --device device-id
```

These workflows are from `11.0-preview.6`.

## MSBuild and SDK evaluation

Visual Studio 2026 and `msbuild.exe` can run .NET-built MSBuild tasks through
`TaskHostFactory`:

```xml
<UsingTask TaskName="MyTask"
           AssemblyFile="path\to\MyTask.dll"
           Runtime="NET"
           TaskFactory="TaskHostFactory" />
```

This path runs out of process and does not support task Host Objects. A
conditional second `UsingTask` without the factory can preserve in-process
execution under Core MSBuild.

Target-framework `DefineConstants` are not available during evaluation
(`10.0-guides`). Do not make evaluation-time conditions depend on values
created only after target-framework processing.

When `DOTNET_CLI_USE_MSBUILD_SERVER` is unset, the CLI no longer forces
`MSBUILDUSESERVER=0` (`11.0-preview.6`). Preserve an explicit environment
setting when build-server use must be deterministic.

Any standard `OTEL_EXPORTER_OTLP_*` variable enables the CLI's OTLP exporter,
in addition to the dedicated exporter flag.

## Workloads and code coverage

Workload management defaults to workload-set mode instead of loose manifests
(`10.0-guides`). Keep the workload-set version under source control where
reproducibility matters.

Dynamic native code-coverage instrumentation defaults to `false`. Enable it
explicitly if native dynamic instrumentation is part of the test contract.

## NuGet restore and package validation

Restore and packaging changes from `10.0-guides`:

- `dotnet restore` audits transitive packages.
- A versionless `PackageReference` is an error.
- A direct reference pruned by NuGet raises NU1510.
- `PrunePackageReference` makes direct prunable references private.
- Packages without runtime assets are omitted from `deps.json`.
- HTTP warnings in package list and search are errors.
- Invalid package IDs are errors.
- SHA-1 signing fingerprints are deprecated.
- `NUGET_ENABLE_ENHANCED_HTTP_RETRY` has been removed.

Treat audit and validation failures as part of restore's contract. Do not
expect the removed retry variable to change HTTP behavior.

Additional package compatibility diagnostics in
`11.0-preview.6-compatibility`:

- NU1703 is emitted when a package falls back to deprecated MonoAndroid
  framework assets.
- `NuGet pack` warns with NU5052 for restricted characters in a package ID.
- .NET template-engine packages no longer support `netstandard2.0`, a source
  and binary compatibility break for consumers on that target.

## Microsoft.Testing.Platform

Select Microsoft.Testing.Platform for `dotnet test` in `global.json`:

```json
{
  "test": {
    "runner": "Microsoft.Testing.Platform"
  }
}
```

Alternatively, `DOTNET_TEST_RUNNER` selects `VSTest` or
`Microsoft.Testing.Platform` without editing `global.json`.

MTP controls in `11.0-preview.6` include:

- `--no-dependencies`;
- `--use-current-runtime`;
- exclusion patterns in `--test-modules`;
- two-stage Ctrl+C cancellation;
- live output; and
- MAUI `--device`.

The xUnit template can create a v3 project that defaults to MTP, while NUnit
can opt in explicitly:

```bash
dotnet new xunit --xunit-version v3
dotnet new nunit --test-runner Microsoft.Testing.Platform
```

VSTest no longer brings in `Newtonsoft.Json` transitively
(`11.0-preview.6-compatibility`). Integrations that use it must declare their
own compatible dependency.

## .NET Framework launch compatibility

The SDK no longer sets the Mono launch target for .NET Framework applications
(`11.0-preview.6-compatibility`). Workflows that relied on that inferred
target must specify their launch path explicitly.
