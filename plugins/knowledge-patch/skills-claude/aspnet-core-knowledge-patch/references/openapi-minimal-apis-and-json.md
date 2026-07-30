# OpenAPI, Minimal APIs, and JSON

## Contents

- [OpenAPI document-version defaults](#openapi-document-version-defaults)
- [Transformer migration](#transformer-migration)
- [Generated schema behavior](#generated-schema-behavior)
- [XML documentation and transformer schema generation](#xml-documentation-and-transformer-schema-generation)
- [File response schemas](#file-response-schemas)
- [QUERY operations](#query-operations)
- [Binding and response fidelity](#binding-and-response-fidelity)
- [Endpoint filters after binding failures](#endpoint-filters-after-binding-failures)
- [PipeReader-based JSON input](#pipereader-based-json-input)
- [C# union types](#c-union-types)

## OpenAPI document-version defaults

Generated documents default to OpenAPI 3.1 in 10.0. OpenAPI 3.2 became an
explicit option in 11.0-preview.2, and `Microsoft.AspNetCore.OpenApi` then
updated its `Microsoft.OpenApi` dependency to 3.3.1. That dependency contains
API changes that can require transformer and integration migrations.

Select 3.2 explicitly in that preview:

```csharp
builder.Services.AddOpenApi(options =>
{
    options.OpenApiVersion =
        Microsoft.OpenApi.OpenApiSpecVersion.OpenApi3_2;
});
```

OpenAPI 3.2 is the generation default in 11.0-preview.6. Pin an earlier
`OpenApiVersion` when downstream tools cannot consume 3.2.

## Transformer migration

OpenAPI.NET 2 changes apply even when the generated document is configured for
OpenAPI 3.0 (10.0):

- OpenAPI entities are interfaces with distinct inline and reference
  implementations.
- Replace `OpenApiSchema.Nullable` with a check for
  `JsonSchemaType.Null`.
- Replace `OpenApiAny` with `JsonNode`.

Account for these changes, plus the later `Microsoft.OpenApi` 3.3.1 dependency
update, when compiling custom document, operation, or schema transformers.

## Generated schema behavior

### Nullability and numeric strings

With OpenAPI 3.1 generation, nullable scalar schemas use a type array that
contains `null`; nullable complex types and collections use `oneOf`
(10.0).

ASP.NET Core's default `JsonNumberHandling.AllowReadingFromString` means
`int` and `long` schemas use a digit pattern without `type: integer`. Configure
number handling as `Strict` when consumers require integer schemas.

### Nullable Minimal API form values

For a complex `[FromForm]` parameter, an empty string posted to a nullable
value-type property binds as `null` instead of producing a parse failure
(10.0).

## XML documentation and transformer schema generation

### XML comments

Set `GenerateDocumentationFile` so the OpenAPI source generator can populate
summaries, remarks, parameter descriptions, return descriptions, and comments
from referenced projects (10.0):

```xml
<GenerateDocumentationFile>true</GenerateDocumentationFile>
```

Minimal API lambdas cannot carry this documentation metadata. Use a documented
method as the endpoint handler when generated comments are required.

### Generate schemas from C# types

Document, operation, and schema transformer contexts expose
`GetOrCreateSchemaAsync` for generating a schema from a C# type
(10.0). Operation and schema contexts also expose `Document`, allowing the
schema to be registered through `AddComponent`.

## File response schemas

### `FileContentResult`

Advertising a `FileContentResult` produces an OpenAPI `type: string`,
`format: binary` response schema in Minimal API and controller applications
(11.0-preview.1).

```csharp
app.MapPost("/download", () =>
    TypedResults.File("file contents"u8.ToArray()))
    .Produces<FileContentResult>(
        contentType: MediaTypeNames.Application.Octet);
```

Controllers can advertise the same response with
`ProducesResponseType<FileContentResult>`.

### Additional concrete file results

Generation also emits binary-string response schemas for `FileStreamResult`,
`FileContentHttpResult`, and `FileStreamHttpResult`
(11.0-preview.4). The endpoint must advertise the concrete result through
`.Produces<T>(contentType: ...)`.

## QUERY operations

Endpoints mapped to the proposed safe, idempotent HTTP `QUERY` method appear
as `query` operations in OpenAPI 3.2 documents (11.0-preview.4). OpenAPI 3.0
and 3.1 documents put them in the Path Item's
`x-oai-additionalOperations` extension.

```csharp
app.MapMethods(
    "/search",
    ["QUERY"],
    (SearchRequest request) => SearchService.Run(request));
```

## Binding and response fidelity

Generated OpenAPI distinguishes transport binding from JSON-body
serialization (11.0-preview.5):

- Non-body enum parameter schemas keep C# member names, matching
  `Enum.TryParse`, even when HTTP JSON uses a naming policy.
- Body schemas continue to use the JSON representation.
- Array component IDs use valid names such as `TodoArray`.
- Multiple `Produces<T>()` calls or `[ProducesResponseType]` attributes for
  one status code remain separate media-type entries or become an `anyOf`
  schema instead of being collapsed.

## Endpoint filters after binding failures

Minimal API filter pipelines run when parameter binding fails, so a filter can
observe the 400 response and replace it (11.0-preview.4). This behavior is
also available in 10.0.8.

In Development, set `RouteHandlerOptions.ThrowOnBadRequest = false` to produce
the 400 response instead of throwing `BadHttpRequestException`.
Non-Development environments already default to `false`.

## PipeReader-based JSON input

MVC, Minimal APIs, and `ReadFromJsonAsync` deserialize through `PipeReader`
(10.0). A custom `JsonConverter` that assumes
`Utf8JsonReader.ValueSpan` contains the full token can lose data when
`HasValueSequence` is true.

Handle both input forms:

```csharp
var span = reader.HasValueSequence
    ? reader.ValueSequence.ToArray()
    : reader.ValueSpan;
```

As a temporary compatibility measure, set the
`Microsoft.AspNetCore.UseStreamBasedJsonParsing` AppContext switch to `true`.

## C# union types

Preview C# unions work without ASP.NET-specific configuration as JSON request
bodies and results in Minimal APIs, MVC, and Razor Pages; in SignalR's JSON
protocol; as Blazor component parameters; in JavaScript interop; and in
persisted component state (11.0-preview.6).

OpenAPI emits `anyOf` without a discriminator:

```csharp
public record class Dog(string Name);
public record class Cat(int Lives);
public union Pet(Dog, Cat);

app.MapGet("/pet", Pet () => new Dog("Rex"));
```

Current limitations:

- Non-JSON binding is unsupported.
- SignalR MessagePack and Newtonsoft.Json are unsupported.
- Current Swashbuckle and NSwag integrations do not recognize unions.
