# Serialization, Hosting, Configuration, and Data

Use this reference for JSON and XML contracts, configuration binding,
validation, background services, logging and telemetry integration, caching,
and EF Core. Relevant source batches are `10.0-guides`, `10.0`,
`11.0-preview.6-compatibility`, and `11.0-preview.6`.

## System.Text.Json compatibility

### Property-name conflicts

`System.Text.Json` checks for property-name conflicts (`10.0-guides`).
Contracts that previously produced ambiguous or colliding JSON member names
can now fail during metadata construction or serialization. Check naming
policies, inherited members, and explicit names when upgrading.

### Duplicate properties and strict mode

Set `AllowDuplicateProperties = false` to make serializers, `JsonObject`,
dictionaries, and `JsonDocument` reject duplicate property names with
`JsonException`:

```csharp
var options = new JsonSerializerOptions
{
    AllowDuplicateProperties = false
};
var value = JsonSerializer.Deserialize<Model>(json, options);
```

The `JsonSerializerOptions.Strict` preset additionally:

- disallows unmapped members;
- keeps property binding case-sensitive;
- enforces nullable annotations; and
- enforces required constructor parameters.

Strict mode is useful at trust boundaries but can turn previously tolerated
input into an error.

### Source-generated reference handling

`JsonSourceGenerationOptionsAttribute.ReferenceHandler` accepts a
`JsonKnownReferenceHandler`, allowing a generated context to preserve
references instead of throwing on cycles:

```csharp
[JsonSourceGenerationOptions(
    ReferenceHandler = JsonKnownReferenceHandler.Preserve)]
[JsonSerializable(typeof(Node))]
partial class AppJsonContext : JsonSerializerContext;
```

This source-generation feature is from batch `10.0`.

## JSON contracts and naming

In `11.0-preview.6`, `System.Text.Json` adds:

- `JsonNamingPolicy.PascalCase`;
- per-member `JsonNamingPolicyAttribute`;
- type-level `JsonIgnoreAttribute` defaults;
- built-in F# discriminated-union support; and
- reflection-based and source-generated C# union contracts.

Customize C# union contracts with `JsonUnionAttribute`, `JsonUnionCaseInfo`,
and type classifiers.

`SerializeAsyncEnumerable` can write to a `PipeWriter`. Passing
`topLevelValues: true` emits newline-delimited top-level values rather than a
single JSON array. Ensure the consumer expects that framing.

## CBOR and XML compatibility

`CborReader` and `CborWriter` enforce a default maximum nesting depth in
`11.0-preview.6-compatibility`. Deep documents can be rejected even when the
application did not configure an explicit limit.

`XmlSerializer` no longer ignores properties marked with
`ObsoleteAttribute` (`10.0-guides`). Such properties can become part of the
serialized contract; apply an XML-specific ignore mechanism if exclusion is
required.

## Configuration binding

Configuration preserves null values in `10.0-guides`. Code that previously
observed a missing or transformed value may now receive an explicit null.

`ConfigurationIgnoreAttribute` excludes an individual model property from
configuration binding (`11.0-preview.6`).

Trim-related `DynamicallyAccessedMembers` annotations were removed from
trim-unsafe `Microsoft.Extensions.Configuration` code. Do not interpret the
absence of those annotations as evidence that reflection-based binding is
trim-safe.

The ICU override variable is `DOTNET_ICU_VERSION_OVERRIDE`.

## Synchronous and asynchronous validation

DataAnnotations in `11.0-preview.6` adds:

- `AsyncValidationAttribute`;
- `IAsyncValidatableObject`; and
- asynchronous `Validator` methods.

Options validation has matching asynchronous support, including
`IAsyncStartupValidator`. Use the asynchronous path when validation performs
I/O or otherwise cannot complete synchronously, and ensure startup awaits the
result.

## Background services and host lifecycle

All of `BackgroundService.ExecuteAsync` runs as a `Task`
(`10.0-guides`). Code that depended on a synchronous prefix running inline
before the first await must account for the scheduling change.

In `11.0-preview.6-compatibility`, failures from a `BackgroundService`
propagate from `IHost.RunAsync` and `IHost.StopAsync`. Callers should catch,
log, or otherwise handle service failures at those host lifecycle boundaries.

## Logging, cache metrics, and tracing

`ProviderAliasAttribute` moved to
`Microsoft.Extensions.Logging.Abstractions` (`10.0-guides`). Update assembly
and package assumptions when reflecting over or referencing the attribute.

With `TrackStatistics=true`, `MemoryCache` publishes these instruments:

- `dotnet.cache.requests`;
- `dotnet.cache.evictions`;
- `dotnet.cache.entries`; and
- `dotnet.cache.estimated_size`.

Injecting `IMeterFactory` scopes instruments per cache rather than using the
shared process meter.

`AddTracing` configures source- and operation-specific rules for enabling or
disabling `Activity` creation without manually building listeners:

```csharp
builder.Services.AddTracing(t =>
    t.DisableTracing(
        sourceName: "MyCompany.Orders",
        operationName: "HealthCheck"));
```

For schema URLs, serialized events and links, sampling, and trace-context
defaults, see [runtime-and-io.md](runtime-and-io.md).

## EF Core query filters

EF Core 10 supports multiple named query filters on an entity type. Individual
filters can be disabled selectively rather than disabling every filter for
that entity.

Use stable names that communicate policy intent, especially when one filter
implements tenant isolation and another implements soft deletion. Selective
disablement should remain explicit at the query site.
