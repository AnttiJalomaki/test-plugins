# Forms, Validation, and Persistence

## Contents

- [Recursive source-generated validation](#recursive-source-generated-validation)
- [Blazor form validation](#blazor-form-validation)
- [Minimal API validation and form binding](#minimal-api-validation-and-form-binding)
- [Localized labels and errors](#localized-labels-and-errors)
- [Persistent component state](#persistent-component-state)
- [TempData during server-side rendering](#tempdata-during-server-side-rendering)
- [Session-backed parameters](#session-backed-parameters)

## Recursive source-generated validation

Call `AddValidation`, declare the model in a C# file rather than a Razor file,
and apply `[ValidatableType]` to the root to validate nested objects and
collections without reflection (10.0).

```csharp
builder.Services.AddValidation();

[ValidatableType]
public sealed class Order
{
    [Required]
    public string? Number { get; set; }
}
```

Apply `[SkipValidation]` to a property or type to stop recursive validation.
For a model defined in another assembly, both the model assembly and the
application must call `AddValidation`.

## Blazor form validation

### Client-side validation during static SSR

A static SSR form containing `DataAnnotationsValidator` validates immediately
in Blazor's client code without a server round trip (11.0-preview.5). The
behavior is enabled by default for enhanced and non-enhanced forms, and .NET
data annotations remain the source of validation rules.

```razor
<EditForm Model="Model" FormName="registration">
    <DataAnnotationsValidator />
</EditForm>
```

### Asynchronous validators

`EditForm` awaits asynchronous validators in every render mode
(11.0-preview.5). Interactive validators register field tasks with
`EditContext.AddValidationTask`; a superseded task is canceled.
`IsValidationPending` and `IsValidationFaulted` expose state, and
`ValidateAsync` awaits completion.

```csharp
var cts = new CancellationTokenSource();
editContext.AddValidationTask(
    field,
    CheckAsync(field, value, cts.Token),
    cts);
await editContext.ValidateAsync();
```

Built-in asynchronous DataAnnotations validation was not included with this
Blazor preview, so register custom asynchronous work explicitly.

## Minimal API validation and form binding

### Nullable form properties

For a complex `[FromForm]` parameter, an empty string posted to a nullable
value-type property binds as `null` rather than causing a parse failure
(10.0).

### Asynchronous DataAnnotations

After `AddValidation()`, Minimal API validation executes
`AsyncValidationAttribute` and `IAsyncValidatableObject`
(11.0-preview.6). The latter returns
`IAsyncEnumerable<ValidationResult>`. An async-only implementation must still
implement the corresponding synchronous member; throw from that member so a
synchronous validation path cannot silently skip the rule.

### Custom resolver migration

Custom validation resolvers must replace `IValidatableInfo` with the
specialized type-, parameter-, and property-specific interfaces and report
errors through `ValidateContext.AddValidationError`
(11.0-preview.6). Ordinary `[ValidatableType]` plus `AddValidation()` usage
does not require this migration.

## Localized labels and errors

### Metadata-aware labels

The `Label` component obtains text from `[Display]`, then `[DisplayName]`, and
finally the property name (11.0-preview.1). It can wrap the control or remain
separate. In the separate form, built-in inputs automatically generate a
matching `id`.

```razor
<Label For="() => model.Name" />
<InputText @bind-Value="model.Name" />
```

### Display names outside labels

`DisplayName` renders localized property metadata outside a label. It checks
`[Display]` before `[DisplayName]`, then falls back to the property name
(11.0-preview.1).

```razor
<DisplayName For="() => product.Price" />
```

### Resource-based error and display names

Blazor and Minimal APIs can localize validation errors and display names from
assembly RESX resources (11.0-preview.5). Attribute `ErrorMessage` and
`Display.Name` values act as lookup keys. Supply a custom
`IStringLocalizerFactory` or `ErrorMessageKeyProvider` for another lookup
strategy.

```csharp
builder.Services.AddValidation()
    .AddValidationLocalization<ValidationMessages>();
```

## Persistent component state

### Declarative persistence

Apply `[PersistentState]` to component or service state that must survive
prerendering instead of manually coordinating the full
`PersistentComponentState` service pattern (10.0-migration).

### Serialization and restore controls

Register `PersistentComponentStateSerializer<T>` to replace JSON
serialization for a type (10.0). `[PersistentState]` also supports:

- `AllowUpdates = true` to accept state updates on enhanced-navigation
  refreshes.
- `RestoreBehavior.SkipInitialValue` to suppress restoration during
  prerendering.
- `RestoreBehavior.SkipLastSnapshot` to suppress restoration during
  reconnection.
- `RegisterOnRestoring` for imperative restoration control.

## TempData during server-side rendering

### Cascaded `ITempData`

`AddRazorComponents()` registers TempData and cascades `ITempData` during
server-side rendering (11.0-preview.2). The default cookie provider uses Data
Protection. `SessionStorageTempDataProvider` is available when session-backed
storage is preferable.

`Get` consumes a one-time value; use `Peek` to inspect without consuming and
`Keep` to retain a value after reading.

```razor
@code {
    [CascadingParameter]
    public ITempData? TempData { get; set; }

    private void Save() => TempData!["Message"] = "Saved";
    private string? Read() => TempData?.Get("Message") as string;
}
```

### Parameters supplied from TempData

Apply `[SupplyParameterFromTempData]` to a Blazor SSR component property
(11.0-preview.4). Reading consumes the TempData value; assigning the property
writes the value back for the next request. Set `Name` when the property and
TempData keys differ.

```csharp
[SupplyParameterFromTempData(Name = "Status")]
public string? StatusMessage { get; set; }
```

## Session-backed parameters

`[SupplyParameterFromSession]` reads and writes Blazor SSR component
properties through HTTP session, using `System.Text.Json`
(11.0-preview.5). Changes are saved before the response is sent. Set `Name` to
use a key other than the property name.

Configure session and its backing cache normally:

```csharp
builder.Services.AddDistributedMemoryCache();
builder.Services.AddSession();
builder.Services.AddRazorComponents();

var app = builder.Build();
app.UseSession();
```
