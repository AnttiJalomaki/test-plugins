# OpenAPI, Minimal APIs, and JSON

Use this reference for generated document semantics, OpenAPI.NET transformer migrations, XML
comments, schema creation, and form-model binding. The changes are from batch `10.0`.

## OpenAPI 3.1 schema defaults

Generated documents default to OpenAPI 3.1. Nullable shapes differ by schema kind:

- Nullable scalar schemas use a type array that includes `null`.
- Nullable complex types and collections use `oneOf`.

Do not post-process all nullable schemas into the same representation.

ASP.NET Core's default `JsonNumberHandling.AllowReadingFromString` also affects number schemas.
Schemas for `int` and `long` use a digit pattern without `type: integer` because JSON strings are
accepted. Configure strict number handling when the document must advertise integer-only input:

```csharp
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.NumberHandling = JsonNumberHandling.Strict;
});
```

Keep serializer behavior and the generated contract aligned; changing only the emitted schema
would misdescribe accepted requests.

## OpenAPI.NET 2 transformer migration

OpenAPI entities are interfaces with distinct inline and reference implementations. Update
transformers that constructed or type-tested the earlier concrete entities.

- Replace `OpenApiSchema.Nullable` with a check for `JsonSchemaType.Null`.
- Replace `OpenApiAny` values with `JsonNode`.
- Account for inline and reference implementations when reading or mutating an entity.

These API migrations are necessary even when ASP.NET Core is configured to emit OpenAPI 3.0.
Document format selection does not restore the earlier OpenAPI.NET object model.

## XML comments in generated OpenAPI

Enable XML documentation output in the project:

```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
</PropertyGroup>
```

The OpenAPI source generator can then populate endpoint summaries, remarks, parameter
descriptions, return descriptions, and comments from referenced projects. Minimal API lambdas
cannot carry this XML metadata. Use a documented method as the endpoint handler when generated
documentation is required.

```csharp
/// <summary>Returns an order.</summary>
/// <param name="id">The order identifier.</param>
/// <returns>The matching order.</returns>
static IResult GetOrder(int id) => Results.Ok(/* ... */);

app.MapGet("/orders/{id}", GetOrder);
```

## Generate schemas from transformers

Document, operation, and schema transformer contexts expose `GetOrCreateSchemaAsync`, which
generates an OpenAPI schema from a C# type. Operation and schema contexts also expose `Document`.
Use the document with `AddComponent` when a generated schema must be registered for reuse.

## Nullable properties in complex form models

For a complex `[FromForm]` parameter, an empty string posted to a nullable value-type property
binds to `null` rather than failing to parse. This is a binding change, not an instruction to
accept missing domain values. Apply validation separately when the property is required by the
application.

```csharp
public sealed class SearchForm
{
    public int? Page { get; set; }
}

app.MapPost("/search", ([FromForm] SearchForm form) => Results.Ok(form));
```

An empty `Page` field reaches the handler as `null`.
