# Forms, Validation, Persistent State, and Identity

## Use declarative prerendered-state persistence

During the `10.0-migration`, components and services can annotate state with
`[PersistentState]` instead of implementing the more involved common pattern
around `PersistentComponentState`.

## Upgrade existing apps to passkeys

Existing Blazor Web Apps can adopt passkey user authentication through the
dedicated ASP.NET Core 10 migration path. Treat this as an upgrade workflow;
do not assume passkey support is restricted to newly generated apps.

## Update Identity redirects when navigation exceptions are disabled

The Blazor Web App template sets:

```xml
<BlazorDisableThrowNavigationException>true</BlazorDisableThrowNavigationException>
```

When an older Individual Accounts app opts into this behavior, update
`Components/Account/IdentityRedirectManager.cs`: remove the
`InvalidOperationException` thrown by `RedirectTo` and remove all five
`[DoesNotReturn]` attributes.

## Enable recursive source-generated validation

In `10.0`, call `AddValidation`, keep the model in a C# file rather than a
Razor file, and annotate the root model with `[ValidatableType]`. Nested
objects and collections are then validated without reflection.

```csharp
builder.Services.AddValidation();

[ValidatableType]
public sealed class Order
{
    [Required]
    public string? Number { get; set; }
}
```

Use `[SkipValidation]` to exclude a property or an entire type. If the model
comes from another assembly, both that assembly and the application must call
`AddValidation` so the required generated metadata is available.

## Control persistent component state

Register `PersistentComponentStateSerializer<T>` to replace JSON
serialization for a state type. `[PersistentState]` supports:

- `AllowUpdates = true` to accept state updates during enhanced-navigation
  refreshes.
- `RestoreBehavior.SkipInitialValue` to skip restoration during prerendering.
- `RestoreBehavior.SkipLastSnapshot` to skip the last snapshot during
  reconnection.

Use `RegisterOnRestoring` when restoration requires imperative control.

## Bind empty nullable form values as null

For a complex Minimal API `[FromForm]` parameter, an empty string posted to a
nullable value-type property binds as `null`. It no longer produces a parse
failure, so remove workarounds or assertions that expect the old error.
