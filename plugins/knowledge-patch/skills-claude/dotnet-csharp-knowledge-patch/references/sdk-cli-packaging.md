# SDK, CLI, Packaging, Testing, and Containers

## CLI compatibility (`10.0-guides`)

- `--interactive` defaults to `true` in user scenarios. Pass an explicit value
  in automation that must never prompt.
- CLI output unrelated to the command result is written to standard error.
  `dotnet watch` also logs to standard error. Parse stdout and stderr according
  to their roles.
- `dotnet new sln` creates SLNX by default. Request the older format explicitly
  when a consumer cannot read SLNX.
- `dotnet package list` performs a restore.
- `dotnet tool install --local` creates a tool manifest when none exists.

## SDK, workload, and tool packaging compatibility (`10.0-guides`)

- .NET tool packaging creates runtime-identifier-specific packages.
- Workload management defaults to workload-set mode rather than loose
  manifests.
- Target-framework `DefineConstants` values are unavailable during evaluation.
  Do not use them to control evaluation-time project structure.
- Dynamic native code-coverage instrumentation defaults to `false`.
- Double quotes in file-level directives are rejected.
- `dnx` scripts bypass `global.json` SDK selection. Select and validate their
  SDK through the supported script workflow rather than assuming the enclosing
  repository pin applies.
- `dnx.ps1` is no longer included.

## NuGet restore, audit, and pruning (`10.0-guides`)

- `dotnet restore` audits transitive packages.
- A versionless `PackageReference` is an error.
- A direct reference pruned by NuGet raises `NU1510`.
- With `PrunePackageReference`, direct prunable references become private.
- Packages without runtime assets are omitted from `deps.json`.
- HTTP warnings in package list or search are errors.
- Invalid package IDs are errors.
- SHA-1 signing fingerprints are deprecated.
- `NUGET_ENABLE_ENHANCED_HTTP_RETRY` was removed. Remove it from build and CI
  environments instead of relying on it to change retry behavior.

## Tool execution and packaging (`10.0`)

### One-shot tool execution

`dotnet tool exec` downloads and runs a tool without installing it. It selects
the latest version unless the package is written as `package@version`, prompts
before a new download, and honors the version from a nearby local tool
manifest.

```bash
dotnet tool exec --source ./artifacts/package dotnetsay@0.1.0 "Hello"
```

### Portable fallback for platform-specific tools

Include the `any` RID alongside platform RIDs to produce a
framework-dependent, platform-agnostic fallback for systems that lack a more
specific tool binary.

```xml
<RuntimeIdentifiers>linux-x64;win-x64;any</RuntimeIdentifiers>
```

## CLI integration and command forms (`10.0`)

Every CLI command accepts `--cli-schema` and emits a JSON description of its
arguments, options, and subcommands.

```bash
dotnet clean --cli-schema
```

Noun-first aliases coexist with the older verb-first commands:

- `dotnet package add|list|remove`
- `dotnet reference add|list|remove`

`dotnet completions script` generates native completion scripts for Bash,
Fish, Nushell, PowerShell, and Zsh.

```bash
dotnet completions script bash
```

## File-based applications (`10.0`)

`dotnet publish app.cs` creates a native executable because file-based apps
publish with native AOT by default. Use `#:property PublishAot=false` when a
dependency is incompatible. File-based apps accept `#:project` references and
support executable, extensionless files with shebang lines.

```csharp
#!/usr/bin/env dotnet
#:project ../ClassLib/ClassLib.csproj
#:property PublishAot=false
Console.WriteLine(new ClassLib.Greeter().Greet());
```

## Container publishing (`10.0`)

Console projects can run `dotnet publish /t:PublishContainer` without setting
`EnableSdkContainerSupport`. `ContainerImageFormat` explicitly chooses
`Docker` or `OCI`; set it instead of inheriting a format from the base image or
multi-architecture arrangement.

```xml
<PropertyGroup>
  <ContainerImageFormat>OCI</ContainerImageFormat>
</PropertyGroup>
```

## Microsoft.Testing.Platform (`10.0`)

`dotnet test` can use Microsoft.Testing.Platform when `global.json` selects the
runner.

```json
{
  "test": {
    "runner": "Microsoft.Testing.Platform"
  }
}
```
