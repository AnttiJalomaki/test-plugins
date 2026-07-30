# Languages and runtimes

This reference includes `components-2025`, `release-notes-117`,
`3.145.0-3.159.0`, `3.160.0-3.181.0`, `3.182.0-3.198.0`,
`3.199.0-3.214.0`, `3.214.1-3.228.0`, `3.229.0-3.248.0`, and
`3.249.0-3.254.0`.

## Node.js and TypeScript

The effective Node.js SDK minimum is Node.js 22. Earlier guidance raised it to
20, added Node.js 24 support, and changed the SDK target from ES2016 to ES2020;
later test coverage dropped Node.js 20 while adding 26. Do not retain 20 as the
supported minimum. TypeScript 6 is accepted as a peer dependency, and generated
projects use `nodenext` for both `module` and `moduleResolution`
(`3.160.0-3.181.0`, `3.229.0-3.248.0`, `3.249.0-3.254.0`).

The Node.js SDK supports pnpm 11. A project's `production` runtime option in
`Pulumi.yaml` makes `pulumi install` use package-manager production mode and
skip `devDependencies` (`3.249.0-3.254.0`).

Node.js package handling can use Bun and configures ESM automatically unless
`--import` or `--require` is explicitly supplied. Entrypoint discovery respects
`package.exports` (`3.182.0-3.198.0`). This package-manager choice is distinct
from the native Bun runtime.

## Bun runtime

The bundled `pulumi-language-bun` plugin runs programs, plugins, debuggers, and
policy packs with Bun (`3.214.1-3.228.0`). Selecting Bun as the Node.js package
manager does not select this language runtime.

## Python

### Toolchains and compatibility

Dynamic providers support Poetry and uv. Pulumi discovers uv plugins/packages,
and `RunPlugin` uses a virtual environment by default
(`3.145.0-3.159.0`). Python projects detect their toolchain from lockfiles and
read Poetry and uv lockfiles when resolving dependencies
(`3.214.1-3.228.0`).

Python 3.14 is supported and uses `grpcio>=1.75.1`. For uv and Poetry projects,
`pulumi new --generate-only` creates `pyproject.toml`
(`3.199.0-3.214.0`).

### Async entrypoints

`pulumi.run` supports a natively awaited program entrypoint, which may return
stack outputs (`3.249.0-3.254.0`).

### Provider and component APIs

Experimental component-provider helpers live under
`pulumi.provider.experimental.component`; the general provider API occupies
`pulumi.provider.experimental.provider`. Component providers can run without a
bootstrap (`3.160.0-3.181.0`).

Python component providers support resource references, enum inference, and
enum references. `@pulumi.type_token` and static `pulumi_type` expose class type
tokens (`3.160.0-3.181.0`). Python providers can set component versions
(`3.199.0-3.214.0`).

Python resource and error hooks have decorator forms, and `Output.recover`
handles output-resolution exceptions (`3.229.0-3.248.0`). Missing
`StackReference` outputs no longer throw (`3.199.0-3.214.0`).

## Go

### Version targets

Generated Go programs and the SDK moved from Go 1.23
(`3.160.0-3.181.0`) to Go 1.25. Automation API supports Go 1.26
(`3.214.1-3.228.0`). Use the later targets when regenerating code.

### Deferred outputs

The SDK and generated programs support deferred outputs, letting code establish
an output handle before assigning the value that resolves it
(`3.145.0-3.159.0`).

### Property and workspace APIs

`property.Value` is immutable. `property.Path` and `property.Map.Delete`
support structured lookup and removal. `workspace.GetPluginInfo`,
`workspace.GetPluginPath`, and APIs that create `plugin.Context` accept
`context.Context` (`3.160.0-3.181.0`).

### Components, tests, and policy

Cross-language Go components use the `pulumi-go-provider` v1 inference builder
and `provider.Run` (`components-2025`). Mock tests can retrieve registered
resources and inspect `GetCurrentExportMap` (`3.182.0-3.198.0`).

Go has an experimental Policy as Code SDK, and Automation API preview/up
options can carry policy packs (`3.160.0-3.181.0`). Policy code receives a
Pulumi `Context` (`3.182.0-3.198.0`).

## Java

The Java SDK reached 1.0 with the complete Pulumi programming API, stronger
type safety, parity with other supported languages, and the Java LTS releases
current at launch (`release-notes-117`). Cross-language components start
`com.pulumi.provider.internal.ComponentProviderHost` with a package to scan
(`components-2025`).

## .NET

Cross-language .NET components return
`Pulumi.Experimental.Provider.ComponentProviderHost.Serve(args)` from
`Program.Main` (`components-2025`). Generated programs call
`RequirePulumiVersion` for project CLI constraints (`3.214.1-3.228.0`).

Provider tooling should migrate from
`github.com/pulumi/pulumi/pkg/v3/codegen/dotnet` to
`github.com/pulumi/pulumi-dotnet/pulumi-language-dotnet/v3/codegen`; the former
package is deprecated and scheduled for removal (`3.214.1-3.228.0`).

## Windows executable resolution

CLI executable lookup on Windows searches `.cmd` and `.ps1` extensions
(`3.199.0-3.214.0`).

## Cross-language scalar methods and cancellation

Provider methods can return scalar values. Go and Python SDK generation and
Node.js program generation support these returns; Node.js uses `callSingle`
(`3.160.0-3.181.0`).

Node.js and Python providers expose cancel handlers. Bun, Go, Node.js, and
Python propagate cancellation to language-host runs, and hosts issue `Cancel`
while closing plugins (`3.229.0-3.248.0`).
