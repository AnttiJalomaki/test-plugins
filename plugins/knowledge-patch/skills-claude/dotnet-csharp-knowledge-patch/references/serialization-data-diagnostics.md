# Serialization, data, validation, and diagnostics

## JSON compatibility checks

In .NET 10, `System.Text.Json` checks for property-name conflicts
(`10.0-guides`). Audit naming policies, explicit names, inheritance, and
source-generated contracts for collisions.

Set `AllowDuplicateProperties = false` to make serializers, `JsonObject`,
dictionaries, and `JsonDocument` reject duplicate names with `JsonException`
(`10.0`).

```csharp
var options = new JsonSerializerOptions
{
    AllowDuplicateProperties = false
};
var value = JsonSerializer.Deserialize<Model>(json, options);
```

The `JsonSerializerOptions.Strict` preset also rejects unmapped members, keeps
case-sensitive property binding, and enforces nullable annotations and
required constructor parameters.

## Source generation and reference preservation

`JsonSourceGenerationOptionsAttribute.ReferenceHandler` accepts a
`JsonKnownReferenceHandler`. A generated context can therefore preserve
references instead of throwing for a cyclic object graph.

```csharp
[JsonSourceGenerationOptions(
    ReferenceHandler = JsonKnownReferenceHandler.Preserve)]
[JsonSerializable(typeof(Node))]
partial class AppJsonContext : JsonSerializerContext;
```

## JSON contracts and streaming

.NET 11 Preview 6 adds the following (`11.0-preview.6`):

- `JsonNamingPolicy.PascalCase`.
- Per-member `JsonNamingPolicyAttribute`.
- Type-level `JsonIgnoreAttribute` defaults.
- Built-in F# discriminated-union support.
- Reflection and source-generated C# union contracts customizable through
  `JsonUnionAttribute`, `JsonUnionCaseInfo`, and type classifiers.
- `SerializeAsyncEnumerable` output to a `PipeWriter`.
- A `topLevelValues: true` mode that emits newline-delimited top-level values
  instead of a JSON array.

## XML contracts

`XmlSerializer` no longer ignores properties marked with
`ObsoleteAttribute` in .NET 10. Such properties can enter the serialized
contract; explicitly exclude them if that is the intended wire shape.

## CBOR depth

`CborReader` and `CborWriter` enforce a default maximum nesting depth in .NET
11 Preview 6 (`11.0-preview.6-compatibility`). Deep CBOR can now be rejected
even when the application did not configure an explicit limit.

## Configuration binding and validation

.NET 11 Preview 6 adds `ConfigurationIgnoreAttribute` for excluding individual
properties from binding. DataAnnotations adds:

- `AsyncValidationAttribute`.
- `IAsyncValidatableObject`.
- Asynchronous `Validator` methods.
- Asynchronous options validation.
- `IAsyncStartupValidator`.

Use the asynchronous path end to end; do not block asynchronous validators
inside synchronous validation.

## EF Core query filters

EF Core 10 supports multiple named query filters on one entity type. Disable
an individual named filter when needed rather than disabling all filters for
that entity.

## Telemetry schemas and sampling

In .NET 10, `ActivitySource` and `Meter` can carry a telemetry schema URL.
`ActivitySourceOptions` is the multi-option construction path. Out-of-process
`Activity` serialization includes events and links.

EventSource trace aggregators can rate-limit root activities with a filter
such as:

```text
[AS]*/-ParentRateLimitingSampler(100)
```

Also revalidate sampling after upgrading: `ActivitySource.CreateActivity` and
`StartActivity` sampling behavior changed, and the default trace-context
propagator is the W3C standard.

## Cache metrics and tracing rules

With `TrackStatistics=true`, `MemoryCache` publishes these instruments in .NET
11 Preview 6:

- `dotnet.cache.requests`
- `dotnet.cache.evictions`
- `dotnet.cache.entries`
- `dotnet.cache.estimated_size`

Injecting `IMeterFactory` scopes them per cache instead of using the shared
process meter.

`AddTracing` configures source-specific and operation-specific `Activity`
enable or disable rules without manually building listeners.

```csharp
builder.Services.AddTracing(t =>
    t.DisableTracing(
        sourceName: "MyCompany.Orders",
        operationName: "HealthCheck"));
```
