# Serialization, Hosting, Configuration, and Data

This topic reference consolidates work from `10.0-guides` and `10.0`.

## Hosting and Background Services

All of `BackgroundService.ExecuteAsync` now runs as a `Task`. The method's
initial code is no longer a special synchronous prefix. Revisit startup order,
exception propagation, and tests that expected work before the first `await`
to run inline during service startup.

Keep long-running work asynchronous and make any required synchronous startup
contract explicit through the host lifecycle rather than depending on the old
execution split.

## Configuration Null Values

Configuration preserves null values. A null can therefore remain distinct
from a missing key or an empty string as providers are combined and values are
bound. Audit fallback logic and binders that previously assumed nulls would be
discarded or normalized away.

## Logging and Trim Annotations

`ProviderAliasAttribute` lives in
`Microsoft.Extensions.Logging.Abstractions`. Update namespace, assembly, and
package assumptions in provider code and reflection-based discovery.

Trim-related `DynamicallyAccessedMembers` annotations were removed from
trim-unsafe `Microsoft.Extensions.Configuration` code. Do not treat the absence
of a warning as proof that reflection-heavy binding is trim-safe; preserve the
required members explicitly or use a trim-compatible binding path.

The ICU override environment variable is `DOTNET_ICU_VERSION_OVERRIDE`.
Replace older names in runtime configuration, container definitions, and
launch scripts.

## Property-Name Conflict Validation

`System.Text.Json` checks for property-name conflicts in a type's effective
JSON contract. Naming policies, attributes, inheritance, and case rules can
make distinct CLR members collide. Resolve collisions deliberately instead of
expecting one property to silently win.

## Duplicate-Safe and Strict JSON

Set `AllowDuplicateProperties` to false to reject duplicate JSON object names
with `JsonException`:

```csharp
var options = new JsonSerializerOptions
{
    AllowDuplicateProperties = false
};
var value = JsonSerializer.Deserialize<Model>(json, options);
```

The setting applies across serializers, `JsonObject`, dictionaries, and
`JsonDocument` processing.

`JsonSerializerOptions.Strict` is a broader preset. It rejects unmapped
members, keeps property binding case-sensitive, enforces nullable annotations,
and enforces required constructor parameters. Adopt it as a contract change:
validate existing payloads and clients before enabling it at a shared boundary.

## Source-Generated Reference Preservation

`JsonSourceGenerationOptionsAttribute.ReferenceHandler` accepts a
`JsonKnownReferenceHandler`. Generated contexts can therefore preserve object
references and cycles rather than throwing:

```csharp
[JsonSourceGenerationOptions(
    ReferenceHandler = JsonKnownReferenceHandler.Preserve)]
[JsonSerializable(typeof(Node))]
partial class AppJsonContext : JsonSerializerContext;
```

Consumers must understand the reference metadata emitted by preserve mode.

## XML Serialization of Obsolete Properties

`XmlSerializer` no longer ignores properties solely because they carry
`ObsoleteAttribute`. Those properties can enter the serialized schema and
payload. Use explicit XML serialization controls when an obsolete member must
remain excluded, and verify backward compatibility for generated contracts.

## Named EF Core Query Filters

EF Core 10 supports several named query filters on one entity type. A caller
can selectively disable one named filter rather than disabling every filter
for that entity.

Give filters stable, purpose-oriented names, especially for concerns such as
soft deletion and tenancy. Disable only the policy required by the operation
so unrelated protections remain active.
