# OpenAPI, HTTP APIs, and JSON

These API and serialization behaviors apply to ASP.NET Core `10.0`.

## OpenAPI 3.1 schema defaults

Generated documents default to OpenAPI 3.1. Nullable scalar schemas use a type
array that contains `null`; nullable complex types and collections use
`oneOf`.

The default `JsonNumberHandling.AllowReadingFromString` behavior means `int`
and `long` schemas can use a digit pattern without `type: integer`. Configure
number handling as `Strict` when consumers require integer schemas.

## Migrate transformers to OpenAPI.NET 2

OpenAPI entities are interfaces with separate inline and reference
implementations. Transformer code must work with that model. Also replace:

- `OpenApiSchema.Nullable` with checks for `JsonSchemaType.Null`.
- `OpenApiAny` with `JsonNode`.

These changes are required even when the generated document targets OpenAPI
3.0 rather than 3.1.

## Populate OpenAPI from XML comments

Set `GenerateDocumentationFile` so the OpenAPI source generator can populate
summaries, remarks, parameter descriptions, return descriptions, and comments
from referenced projects:

```xml
<GenerateDocumentationFile>true</GenerateDocumentationFile>
```

Minimal API lambdas cannot carry this documentation metadata. Use a documented
method as the endpoint handler when generated endpoint documentation matters.

## Generate schemas inside transformers

Document, operation, and schema transformer contexts expose
`GetOrCreateSchemaAsync` for creating a schema from a C# type. Operation and
schema transformer contexts also expose `Document`; use it with `AddComponent`
to register the generated schema in the document.

## Support multi-segment JSON tokens

MVC, Minimal APIs, and `ReadFromJsonAsync` deserialize through `PipeReader`.
Custom converters that only inspect `Utf8JsonReader.ValueSpan` can lose data
when `HasValueSequence` is `true`.

```csharp
var span = reader.HasValueSequence
    ? reader.ValueSequence.ToArray()
    : reader.ValueSpan;
```

Update converters to read `ValueSequence`. As a temporary fallback, set the
`Microsoft.AspNetCore.UseStreamBasedJsonParsing` AppContext switch to `true`.
