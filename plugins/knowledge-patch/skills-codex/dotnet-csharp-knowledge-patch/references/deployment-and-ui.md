# Deployment, Containers, Native Output, and UI

This reference combines deployment and desktop compatibility work from
`10.0-guides` and `10.0`.

## Container Base Distribution

Default .NET 10 container images use Ubuntu. Any build or runtime layer that
depends on the previous distribution's package manager, package names,
filesystem paths, native library versions, user setup, or shell tools must
either adapt or pin a compatible base image.

Do not infer distribution compatibility from the .NET tag alone. Resolve the
actual base image and exercise native dependencies in the produced container.

## Publishing Console Projects as Containers

Console projects can invoke container publishing directly:

```bash
dotnet publish /t:PublishContainer
```

`EnableSdkContainerSupport` is no longer required for this scenario. Remove
unnecessary compatibility properties when they obscure the active SDK
behavior.

## Selecting Docker or OCI Output

Use `ContainerImageFormat` to select the image format explicitly:

```xml
<PropertyGroup>
  <ContainerImageFormat>OCI</ContainerImageFormat>
</PropertyGroup>
```

Valid choices include `Docker` and `OCI`. Without an explicit value, the
effective format can depend on the base image and whether the publish is
multi-architecture. Pin it when registries, signing, inspection, or deployment
systems require one format.

## Publishing File-Based Applications

Publishing a file-based application with `dotnet publish app.cs` produces a
native executable by default through native AOT. Opt out when reflection,
dynamic code, or another dependency is incompatible:

```csharp
#:property PublishAot=false
```

File-based applications also support `#:project` references. An extensionless
file with a `#!/usr/bin/env dotnet` shebang can be executable, which is useful
for source-first utilities. Account for native AOT platform targeting and test
the produced executable on the destination system.

## WPF and Windows Forms Type Collisions

Projects that reference both WPF and Windows Forms must disambiguate
`MenuItem` and `ContextMenu`. Use aliases or fully qualified names at the
boundary instead of relying on an ambiguous `using` set.

## Windows Forms Compatibility

- `HtmlElement.InsertAdjacentElement` has a renamed parameter. Update named
  arguments; positional calls are unaffected by the source-level name change.
- `StatusStrip` defaults to the system render mode. Set a render mode
  explicitly when visual consistency is required.

## System.Drawing Exceptions

Some `System.Drawing` failures now throw `ExternalException` instead of
`OutOfMemoryException`. Narrow catches and tests to the new failure contract.
Avoid using `OutOfMemoryException` as a generic signal for drawing or image
format failures.

## WPF Validation and Failure Behavior

WPF rejects empty `ColumnDefinitions` and `RowDefinitions`. Remove empty
collections or provide actual definitions rather than relying on permissive
parsing.

Incorrect `DynamicResource` usage can terminate the application. Validate
resource keys, resource types, and lookup placement during startup and in UI
tests; do not assume every malformed dynamic resource degrades to a recoverable
binding warning.
