# SDK, CLI, Build, Restore, and Test

This workflow-oriented reference draws on `10.0-guides` and `10.0`.

## CLI Defaults and Output Contracts

The CLI's `--interactive` option defaults to `true` in user scenarios. Pass an
explicit noninteractive choice in CI or any automation that cannot answer a
prompt.

Output not directly relevant to a command's result goes to standard error.
`dotnet watch` logging also goes to standard error. Scripts should parse
standard output as the result channel while preserving standard error for
diagnostics instead of merging them indiscriminately.

`dotnet new sln` creates SLNX by default. Specify another format when tooling
still requires the older solution representation.

`dotnet package list` performs a restore. Account for network access, audit
failures, and changed assets when treating a list operation as read-only.

`dotnet tool install --local` creates a tool manifest if none exists. Check in
the manifest when it is intended to define repository tooling.

## One-Shot Tool Execution

`dotnet tool exec` downloads and runs a tool without installing it:

```bash
dotnet tool exec --source ./artifacts/package dotnetsay@0.1.0 "Hello"
```

Without `@version`, it selects the latest version. It prompts before a new
download and honors the version in a nearby local tool manifest. Pin a version
and control sources in reproducible or security-sensitive automation.

## Portable Tool Fallback Packages

Tool packaging creates runtime-identifier-specific packages. Include the
`any` RID alongside platform RIDs to provide a framework-dependent,
platform-neutral fallback when no platform-specific binary matches:

```xml
<RuntimeIdentifiers>linux-x64;win-x64;any</RuntimeIdentifiers>
```

The fallback is not a native binary; ensure the destination has a suitable
runtime.

## Machine-Readable CLI and Native Completion

Every CLI command accepts `--cli-schema` and emits a JSON description of its
arguments, options, and subcommands:

```bash
dotnet clean --cli-schema
```

Use this contract for shell integration and tooling instead of scraping help
text.

Noun-first aliases coexist with the older verb-first commands:

- `dotnet package add|list|remove`
- `dotnet reference add|list|remove`

Generate native completion scripts for Bash, Fish, Nushell, PowerShell, or Zsh
with `dotnet completions script <shell>`:

```bash
dotnet completions script bash
```

## SDK and Workload Selection

Workload management defaults to workload-set mode rather than loose manifests.
Pin or update the workload set as a unit when reproducibility matters.

Target-framework `DefineConstants` values are not available during project
evaluation. Do not branch evaluation-time logic on constants that only exist
for compilation.

Dynamic native code-coverage instrumentation defaults to false. Enable it
explicitly when a coverage workflow requires dynamic native instrumentation.

Double quotes in file-level directives are rejected. Use the supported
directive syntax rather than relying on permissive parsing.

`dnx` scripts bypass `global.json` SDK selection. Do not assume a repository's
SDK pin automatically controls them. `dnx.ps1` is no longer shipped, so remove
automation that invokes that script.

## NuGet Restore, Audit, and Pruning

`dotnet restore` audits transitive packages. Surface and triage audit results
in restore workflows rather than assuming only direct references are checked.

- A versionless `PackageReference` is an error.
- Direct references that NuGet can prune produce NU1510.
- `PrunePackageReference` makes direct prunable references private.
- Packages with no runtime assets are omitted from `deps.json`.
- HTTP warnings in package list and search operations are errors.
- Invalid package IDs are errors.
- SHA-1 package-signing fingerprints are deprecated.
- `NUGET_ENABLE_ENHANCED_HTTP_RETRY` has been removed. Delete it from build
  environments and use supported retry behavior.

Review tools that inspect `deps.json`, and decide whether an NU1510 reference
is intentionally direct before removing it.

## .NET Tasks in .NET Framework MSBuild

Visual Studio 2026 and `msbuild.exe` can run tasks built for .NET through
`TaskHostFactory`:

```xml
<UsingTask TaskName="MyTask"
           AssemblyFile="path\to\MyTask.dll"
           Runtime="NET"
           TaskFactory="TaskHostFactory" />
```

The task runs out of process, and task Host Objects are not supported on this
path. When Core MSBuild should keep the task in process, add a conditional
second `UsingTask` without the factory.

## File-Based Applications

`dotnet publish app.cs` produces a native executable because file-based apps
publish with native AOT by default. Disable AOT for incompatible dependencies:

```csharp
#!/usr/bin/env dotnet
#:project ../ClassLib/ClassLib.csproj
#:property PublishAot=false
Console.WriteLine(new ClassLib.Greeter().Greet());
```

File-based apps accept `#:project` references and support executable,
extensionless shebang files. Because `dnx` does not inherit `global.json` SDK
selection, record any required SDK assumptions in the invocation environment.

## Microsoft.Testing.Platform

`dotnet test` uses Microsoft.Testing.Platform when selected in `global.json`:

```json
{
  "test": {
    "runner": "Microsoft.Testing.Platform"
  }
}
```

Make the runner choice repository-visible so local and CI test discovery and
execution agree.
