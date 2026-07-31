# Forms, Validation, and Persistence

Use this reference for Blazor validation, Minimal API form values, prerendered state, enhanced
navigation, and circuit restoration.

## Recursive source-generated Blazor validation

Recursive validation is available in batch `10.0`. Register it with `AddValidation`, define the
model in a C# file rather than a Razor file, and annotate the root type with `[ValidatableType]`.

```csharp
builder.Services.AddValidation();

[ValidatableType]
public sealed class Order
{
    [Required]
    public string? Number { get; set; }

    public ShippingAddress? Address { get; set; }

    public List<OrderLine> Lines { get; set; } = [];
}
```

The generated validator traverses nested objects and collections without reflection.

- Put `[SkipValidation]` on a property or type that must not be traversed.
- If a model lives in another assembly, that assembly and the application must both call
  `AddValidation` so the generated registrations are available.
- Keep the root model discoverable by the source generator; a type declared only inside a Razor
  file does not meet this requirement.

## Nullable values in Minimal API form models

For a complex `[FromForm]` parameter, an empty string posted to a nullable value-type property
binds to `null` instead of producing a parse failure (batch `10.0`). Do not preserve validation
workarounds that translate empty strings solely to avoid the earlier binder failure. Continue to
validate `null` when the domain requires a value.

## Declarative prerendered-state persistence

The migration guidance in batch `10.0-migration` adds `[PersistentState]` as the concise option
for components and services that need to persist state across prerendering. Use it in place of
the imperative `PersistentComponentState` callback pattern when declarative persistence is
sufficient.

```csharp
[PersistentState]
public WeatherForecast[]? Forecasts { get; set; }
```

The imperative service remains appropriate when persistence timing or selection requires custom
control.

## Persistence controls

Batch `10.0` adds controls for serialization, enhanced navigation, and restoration:

- Register `PersistentComponentStateSerializer<T>` to replace JSON persistence for a type.
- Set `[PersistentState(AllowUpdates = true)]` when state should update during an
  enhanced-navigation refresh.
- Use `RestoreBehavior.SkipInitialValue` to avoid restoring the initial prerendered value.
- Use `RestoreBehavior.SkipLastSnapshot` to avoid restoring the last reconnect snapshot.
- Call `RegisterOnRestoring` for imperative restoration logic.

Choose the restoration behavior according to the transition being handled. Prerender-to-
interactive hydration, enhanced navigation, and circuit reconnection are distinct transitions;
a value appropriate for one may be stale in another.

## Server circuit resumption

Server-side Blazor circuit state can survive an extended connection loss or a proactive pause
and resume without discarding unsaved state. A full-page refresh is the boundary: it creates a
new page and does not resume the old circuit. Do not promise persistence across refresh unless
the state is also stored outside the circuit.
