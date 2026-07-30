# Deployment, Containers, Native Output, and UI

Use this reference for container publishing, NativeAOT and native-library
artifacts, runtime configuration precedence, hardware and mobile deployment
requirements, and desktop UI compatibility. It covers relevant items from
`10.0-guides`, `10.0`, `11.0-preview.6-compatibility`, and
`11.0-preview.6`.

## Container base images

Default .NET 10 container images use Ubuntu (`10.0-guides`). Builds that
depend on the prior distribution's package names, filesystem paths, libc
details, or package manager must either:

- pin a compatible base image deliberately; or
- adapt installation and runtime assumptions to Ubuntu.

Rebuild native dependencies and exercise container health checks when changing
the distribution.

## SDK container publishing

Console projects can run:

```bash
dotnet publish /t:PublishContainer
```

They no longer need `EnableSdkContainerSupport` (`10.0`).

Set `ContainerImageFormat` explicitly to avoid inheriting a default affected
by the base image or multi-architecture publishing:

```xml
<PropertyGroup>
  <ContainerImageFormat>OCI</ContainerImageFormat>
</PropertyGroup>
```

Supported explicit values are `Docker` and `OCI`.

In `11.0-preview.6`, the SDK's built-in container publisher can create
multi-architecture images with Podman. Verify that every requested
architecture has matching application and native dependency artifacts.

## File-based publishing and NativeAOT

`dotnet publish app.cs` publishes a file-based application with native AOT by
default (`10.0`). For dependencies that cannot be trimmed or compiled with
NativeAOT, set:

```csharp
#:property PublishAot=false
```

File-based project references, includes, and CLI behavior are detailed in
[sdk-cli-build-and-test.md](sdk-cli-build-and-test.md).

On Unix, NativeAOT native-library outputs use the `lib` prefix
(`11.0-preview.6-compatibility`). Update packaging, loader configuration,
artifact globbing, and deployment manifests to use the new filenames.

## Runtime configuration precedence

In `11.0-preview.6-compatibility`, values under `configProperties` in
`.runtimeconfig.dev.json` override values in `.runtimeconfig.json`. Check both
files when development behavior differs from production, and avoid depending
on the older precedence.

## Hardware and platform baselines

The JIT minimum hardware requirements changed in
`11.0-preview.6-compatibility`. Revalidate older deployment targets before
upgrading instead of assuming a previously supported CPU remains valid.

.NET MAUI requires Android API level 24 or later. Update application minimums,
device matrices, emulators, and store metadata together.

Cryptographic platform requirements, including OpenSSL, DSA, PQC, and macOS
TLS behavior, are in
[security-networking-and-interop.md](security-networking-and-interop.md).

## Windows Forms and WPF

Projects that reference both WPF and Windows Forms must disambiguate
`MenuItem` and `ContextMenu` (`10.0-guides`). Use namespace qualification or
aliases rather than relying on an ambiguous using set.

Other Windows desktop compatibility changes:

- `HtmlElement.InsertAdjacentElement` has a renamed parameter. Update named
  arguments.
- `StatusStrip` defaults to the system render mode. Set the desired render
  mode explicitly when appearance is contractual.
- Some `System.Drawing` failures throw `ExternalException` instead of
  `OutOfMemoryException`. Catch or test the exception representing the actual
  failure boundary.
- WPF rejects empty `ColumnDefinitions` and `RowDefinitions`.
- Incorrect `DynamicResource` usage can terminate the application.

Validate XAML at build/test time where possible, remove empty definitions, and
correct invalid resource references rather than depending on prior tolerance.
