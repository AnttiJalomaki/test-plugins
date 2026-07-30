# OpenAPI, HTTP APIs, and JSON

## OpenAPI document versions and schema shapes

- Generated documents default to OpenAPI 3.1 in 10.0. Nullable scalars use a type array containing `null`; nullable complex types and collections use `oneOf`. With the default `JsonNumberHandling.AllowReadingFromString`, `int` and `long` schemas use a digit pattern without `type: integer`; set number handling to `Strict` when integer schemas are required (10.0).
- `Microsoft.AspNetCore.OpenApi` gained OpenAPI 3.2 generation and updated `Microsoft.OpenApi` to 3.3.1, whose API changes can require transformer and integration migrations. Initially select it with `OpenApiVersion.OpenApi3_2` (11.0-preview.2).

```csharp
builder.Services.AddOpenApi(options =>
    options.OpenApiVersion = Microsoft.OpenApi.OpenApiSpecVersion.OpenApi3_2);
```

- OpenAPI 3.2 later became the default. Pin `OpenApiVersion` to 3.0 or 3.1 explicitly when downstream tooling does not support 3.2 (11.0-preview.6).
- In OpenAPI 3.2, endpoints mapped to the safe, idempotent `QUERY` method appear as `query` operations. In 3.0 and 3.1, they appear in the Path Item `x-oai-additionalOperations` extension (11.0-preview.4).

## Transformers and generated metadata

- OpenAPI.NET 2 models entities as interfaces with separate inline and reference implementations. Replace `OpenApiSchema.Nullable` checks with `JsonSchemaType.Null` checks and replace `OpenApiAny` with `JsonNode`. These transformer migrations are required even when output remains OpenAPI 3.0 (10.0).
- Setting `<GenerateDocumentationFile>true</GenerateDocumentationFile>` lets the source generator populate summaries, remarks, parameter and return descriptions, and comments from referenced projects. Minimal API lambdas cannot carry this metadata; use a documented method as the handler (10.0).
- Document, operation, and schema transformer contexts expose `GetOrCreateSchemaAsync` to create schemas from C# types. Operation and schema contexts also expose `Document`, allowing registration through `AddComponent` (10.0).

## Response schemas and fidelity

- Advertising a `FileContentResult` response produces an OpenAPI `type: string`, `format: binary` schema. Minimal APIs use `.Produces<FileContentResult>(contentType: ...)`; controllers use `[ProducesResponseType<FileContentResult>]` (11.0-preview.1).
- Binary schemas also apply to `FileStreamResult`, `FileContentHttpResult`, and `FileStreamHttpResult` when `.Produces<T>(contentType: ...)` advertises the concrete result (11.0-preview.4).
- Non-body enum parameters retain C# member names even when HTTP JSON uses a naming policy, matching `Enum.TryParse`; request-body schemas still use JSON names. Array component IDs use valid names such as `TodoArray`. Multiple `Produces<T>()` or `[ProducesResponseType]` declarations for one status are retained as media types or an `anyOf` schema instead of being collapsed (11.0-preview.5).
- Preview C# unions work without ASP.NET-specific configuration in Minimal API, MVC, and Razor Pages JSON bodies and results, SignalR JSON, Blazor parameters and JS interop, and persisted component state. OpenAPI emits `anyOf` without a discriminator. Non-JSON binding, SignalR MessagePack, Newtonsoft.Json, and current Swashbuckle/NSwag union recognition are unsupported (11.0-preview.6).

```csharp
public record class Dog(string Name);
public record class Cat(int Lives);
public union Pet(Dog, Cat);
app.MapGet("/pet", Pet () => new Dog("Rex"));
```

## JSON parsing and endpoint execution

- MVC, Minimal APIs, and `ReadFromJsonAsync` deserialize through `PipeReader`. A custom converter must handle `Utf8JsonReader.HasValueSequence`; use `reader.ValueSequence.ToArray()` when true and `reader.ValueSpan` otherwise. Temporarily restore stream parsing with the `Microsoft.AspNetCore.UseStreamBasedJsonParsing` AppContext switch (10.0).
- Minimal API endpoint filters run after parameter binding failures, so a filter can observe the 400 and replace the response. In Development, set `RouteHandlerOptions.ThrowOnBadRequest = false`; other environments already default to `false`. This behavior also shipped in 10.0.8 (11.0-preview.4).
- The source generator emits the `public partial class Program` needed to test apps that use top-level statements. Remove manual declarations from application code (10.0).
