# Serialization, Data, and Diagnostics

## Serialization compatibility (`10.0-guides`)

`System.Text.Json` checks for property-name conflicts. Resolve colliding names
instead of depending on an implicit winner.

`XmlSerializer` no longer ignores properties marked with `ObsoleteAttribute`.
Such properties can enter the serialized contract; exclude them explicitly if
they are not wire data.

## JSON contracts (`10.0`)

### Reference handling in generated contexts

`JsonSourceGenerationOptionsAttribute.ReferenceHandler` accepts a
`JsonKnownReferenceHandler`. A generated context can therefore preserve
references instead of throwing when it encounters a cycle.

```csharp
[JsonSourceGenerationOptions(
    ReferenceHandler = JsonKnownReferenceHandler.Preserve)]
[JsonSerializable(typeof(Node))]
partial class AppJsonContext : JsonSerializerContext;
```

### Duplicate-safe and strict input

Set `AllowDuplicateProperties = false` to make serializers, `JsonObject`,
dictionaries, and `JsonDocument` reject duplicate property names with
`JsonException`.

```csharp
var options = new JsonSerializerOptions
{
    AllowDuplicateProperties = false
};
var value = JsonSerializer.Deserialize<Model>(json, options);
```

The `JsonSerializerOptions.Strict` preset additionally disallows unmapped
members, retains case-sensitive binding, and enforces nullable annotations and
required constructor parameters.

## Diagnostics (`10.0`)

### Telemetry schema URLs and activity serialization

`ActivitySource` and `Meter` can carry a telemetry schema URL.
`ActivitySourceOptions` supplies the constructor path when multiple options are
needed. Out-of-process `Activity` serialization includes events and links, so
consumers should be prepared to receive those collections.

### Rate-limited root-activity sampling

EventSource trace aggregators can cap root activities per second. A filter such
as the following sets the cap to 100:

```text
[AS]*/-ParentRateLimitingSampler(100)
```

## EF Core query filters (`10.0`)

EF Core 10 supports multiple named query filters for an entity type. Disable an
individual named filter when needed instead of disabling every filter attached
to that entity.
