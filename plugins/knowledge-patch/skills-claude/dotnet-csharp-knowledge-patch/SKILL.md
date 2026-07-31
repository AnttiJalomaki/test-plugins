---
name: dotnet-csharp-knowledge-patch
description: .NET and C#
version: null
license: MIT
metadata:
  author: Nevaberry
---


# .NET and C# Knowledge Patch

Use this skill when writing, upgrading, reviewing, or troubleshooting modern
.NET and C# code. Inspect the project SDK, target frameworks, language version,
runtime identifiers, publishing mode, and platform before applying guidance.
Treat the project manifest, lock files, code, and tests as authoritative when
they differ from a general default.

## Reference index

| Reference | Topics |
| --- | --- |
| [language.md](references/language.md) | C# extension blocks, properties, lambdas, partial members, assignments, spans, and Visual Basic compatibility |
| [sdk-cli-packaging.md](references/sdk-cli-packaging.md) | CLI behavior, SDK evaluation, NuGet, tools, MSBuild, testing, file-based apps, and containers |
| [runtime-io.md](references/runtime-io.md) | Core runtime behavior, I/O, metadata, globalization, tensors, and intrinsics |
| [security-networking.md](references/security-networking.md) | Cryptography, certificates, TLS, HTTP, URI, mail, and LDAP |
| [serialization-data-diagnostics.md](references/serialization-data-diagnostics.md) | JSON, XML, tracing, metrics, and EF Core filters |
| [hosting-platform-interop.md](references/hosting-platform-interop.md) | Hosting, configuration, logging, shutdown, native loading, COM, and Windows desktop |

## Start with compatibility changes

### Recheck container and native-library assumptions

- Default .NET 10 container images use Ubuntu. Pin a base image or update
  package names, paths, and package-manager commands that assumed the earlier
  distribution.
- Single-file applications no longer probe their executable directory for
  native libraries.
- `DllImportSearchPath.AssemblyDirectory` searches only the assembly directory.
  Configure an explicit native-library location when another directory is
  required.

### Do not rely on old buffering or shutdown behavior

- `BufferedStream.WriteByte` no longer flushes implicitly. Call `Flush` or
  `FlushAsync` at the durability or visibility boundary.
- The runtime no longer installs default termination-signal handlers. Register
  the signal and shutdown behavior an application actually requires.
- All of `BackgroundService.ExecuteAsync` runs as a `Task`. Audit startup-time
  assumptions and exception flow in hosted services.

### Audit networking and cryptography defaults

- Trimmed publications disable HTTP/3 by default.
- Browser HTTP clients stream responses by default; do not assume the full
  response is buffered.
- Unix cryptography requires OpenSSL 1.1.1 or later, while OpenSSL-backed
  primitives are unavailable on macOS.
- Use `DOTNET_OPENSSL_VERSION_OVERRIDE` and `DOTNET_ICU_VERSION_OVERRIDE`; the
  older override names are not the active controls.
- `MailAddress`, `X500DistinguishedName`, and LDAP `DirectoryControl` parsing
  are stricter. Validate previously accepted inputs during an upgrade.

### Recheck CLI, restore, and packaging automation

- `dotnet new sln` creates SLNX by default.
- Non-command-relevant CLI output and `dotnet watch` logs go to standard error.
  Keep stdout parsers limited to command results.
- `dotnet package list` restores before listing, and `--interactive` defaults
  to true in user scenarios.
- Restore audits transitive dependencies. A versionless `PackageReference`, an
  invalid package ID, and HTTP warnings from package list/search are errors.
- Direct references pruned by NuGet produce `NU1510`; direct prunable
  references become private when `PrunePackageReference` is enabled.
- Tool packages are runtime-identifier-specific. Add the `any` RID when a
  portable framework-dependent fallback is intended.

### Revalidate serialization and desktop behavior

- `System.Text.Json` detects property-name conflicts.
- `XmlSerializer` includes properties marked `ObsoleteAttribute`; exclude them
  explicitly if they must not be part of the contract.
- WPF rejects empty `ColumnDefinitions` and `RowDefinitions`, and invalid
  `DynamicResource` use can terminate the application.
- Projects combining WPF and Windows Forms must qualify ambiguous `MenuItem`
  and `ContextMenu` types.

## Use C# extension blocks

A named receiver declares instance extension members. Omitting the receiver
name permits static extension members. Extension blocks can contain instance or
static properties and methods, as well as operators.

```csharp
public static class SequenceExtensions
{
    extension<T>(IEnumerable<T> source)
    {
        public bool IsEmpty => !source.Any();
    }
}
```

Use the contextual `field` token to validate an auto-property without declaring
a backing field:

```csharp
public string Message
{
    get;
    set => field = value ?? throw new ArgumentNullException(nameof(value));
}
```

If the type already has an identifier named `field`, write `@field` for that
identifier or `this.field` for the existing instance member.

## Apply assignment and lambda syntax precisely

Null-conditional access can be the target of simple or compound assignment.
The right-hand side runs only for a non-null receiver; `++` and `--` are not
supported in this form.

```csharp
customer?.Order = GetCurrentOrder();
customer?.Balance += payment;
```

Implicitly typed lambda parameters can use `scoped`, `ref`, `in`, `out`, and
`ref readonly`. A `params` parameter still requires explicit parameter types.

```csharp
TryParse<int> parse = (text, out result) => int.TryParse(text, out result);
```

Also account for these language rules:

- `nameof(List<>)` returns `"List"` without a type argument.
- User-defined compound-assignment operators can implement dedicated mutation
  behavior instead of falling back to the corresponding binary operator.
- Partial instance constructors and events require one defining declaration
  and one implementing declaration. Only the implementing constructor may use
  `this()` or `base()`; the implementing event provides `add` and `remove`.
- New span conversions and inference can select a different overload. Test
  overload-sensitive calls after changing the language version.

## Prefer explicit JSON contracts

Reject duplicate property names when the input must be unambiguous:

```csharp
var options = new JsonSerializerOptions
{
    AllowDuplicateProperties = false
};
var value = JsonSerializer.Deserialize<Model>(json, options);
```

`JsonSerializerOptions.Strict` also rejects unmapped members, keeps property
binding case-sensitive, and enforces nullable annotations and required
constructor parameters.

Source-generated serializers can preserve references:

```csharp
[JsonSourceGenerationOptions(
    ReferenceHandler = JsonKnownReferenceHandler.Preserve)]
[JsonSerializable(typeof(Node))]
partial class AppJsonContext : JsonSerializerContext;
```

## Select cryptographic APIs by algorithm and capability

Avoid SHA-1-only thumbprint lookup when another digest is intended:

```csharp
var matches = certificates.FindByThumbprint(
    HashAlgorithmName.SHA256, thumbprint);
```

Choose PFX encryption deliberately:

```csharp
byte[] pfx = certificate.ExportPkcs12(
    Pkcs12ExportPbeParameters.Pbes2Aes256Sha256, password);
```

Use the compatible 3DES/SHA-1 preset only when interoperability requires it;
use the AES-256/SHA-256 preset or custom `PbeParameters` otherwise.

Before using `MLKem`, `MLDsa`, or `SlhDsa`, check the type's `IsSupported`
property. Their support depends on OpenSSL 3.5+ or a Windows CNG implementation
with post-quantum support. These types use static generation and import methods,
not `AsymmetricAlgorithm` inheritance; heed `SYSLIB5006` on experimental APIs.

## Use the newer CLI workflows

Run a tool once without installing it:

```bash
dotnet tool exec --source ./artifacts/package dotnetsay@0.1.0 "Hello"
```

Omitting `@version` selects the latest version. A new download prompts first,
and a nearby local tool manifest can determine the version.

Every CLI command can describe itself as JSON for integrations:

```bash
dotnet clean --cli-schema
```

Use noun-first package and reference commands where convenient, and generate
native shell completion with `dotnet completions script <shell>`.

File-based applications publish as native AOT by default. Opt out for
incompatible dependencies and use project directives when composition is
needed:

```csharp
#!/usr/bin/env dotnet
#:project ../ClassLib/ClassLib.csproj
#:property PublishAot=false
Console.WriteLine(new ClassLib.Greeter().Greet());
```

## Verify platform-sensitive features

- Opt macOS clients into TLS 1.3 with the
  `System.Net.Security.UseNetworkFramework` switch or
  `DOTNET_SYSTEM_NET_SECURITY_USENETWORKFRAMEWORK=1`. This is client-only and
  changes more than protocol availability; test buffering, cancellation,
  zero-byte reads, IDN handling, and loss of TLS 1.0/1.1.
- `System.Runtime.Intrinsics.X86.Avx10v2` exists, but its JIT support remains
  disabled by default.
- Tensor slices are non-copying views. Do not treat a slice as an immutable
  snapshot of the underlying storage.
- Console projects can publish containers without
  `EnableSdkContainerSupport`; set `ContainerImageFormat` explicitly to
  `Docker` or `OCI` when reproducible image format matters.

Read the topic reference before making a migration, publishing, security, or
platform decision; the reference files retain edge conditions that the quick
recipes intentionally omit.
