# Forms, Validation, Persistent State, and Identity

## Source-generated validation

- For recursive Blazor validation, call `AddValidation`, define the model in a C# file rather than a Razor file, and mark the root with `[ValidatableType]`. `[SkipValidation]` excludes a property or type. When models live in another assembly, both that assembly and the app must call `AddValidation` (10.0).

```csharp
builder.Services.AddValidation();

[ValidatableType]
public sealed class Order
{
    [Required]
    public string? Number { get; set; }
}
```

- Static SSR forms containing `DataAnnotationsValidator` validate immediately in client-side Blazor without a round-trip. This is on by default for enhanced and non-enhanced forms; .NET data annotations remain the rule source (11.0-preview.5).
- `EditForm` awaits asynchronous validators in every render mode. Interactive validators register field tasks with `EditContext.AddValidationTask`; superseded tasks are canceled. Inspect `IsValidationPending` and `IsValidationFaulted`, and call `ValidateAsync` to await completion. This preview does not include built-in asynchronous DataAnnotations support (11.0-preview.5).
- Blazor and Minimal API validation can localize errors and display names from assembly RESX resources. Attribute `ErrorMessage` and `Display.Name` values become keys. Configure `AddValidation().AddValidationLocalization<T>()`, or provide a custom `IStringLocalizerFactory` or `ErrorMessageKeyProvider` (11.0-preview.5).
- Minimal API validation runs DataAnnotations `AsyncValidationAttribute` and `IAsyncValidatableObject` after `AddValidation()`. `IAsyncValidatableObject` yields `IAsyncEnumerable<ValidationResult>`; an async-only implementation must still implement the synchronous member and should throw there so synchronous validation does not silently skip the rule (11.0-preview.6).
- Custom Minimal API validation resolvers must replace `IValidatableInfo` with the type-, parameter-, and property-specific interfaces and add errors through `ValidateContext.AddValidationError`. Ordinary `[ValidatableType]` and `AddValidation()` callers are unaffected (11.0-preview.6).
- For a complex `[FromForm]` parameter, an empty string posted to a nullable value-type property binds as `null` instead of causing a parse failure (10.0).

## Prerendered and persistent component state

- Mark component or service state with `[PersistentState]` instead of manually coordinating `PersistentComponentState` for ordinary prerender persistence (10.0-migration).
- Register `PersistentComponentStateSerializer<T>` to replace JSON serialization for a type. `[PersistentState(AllowUpdates = true)]` permits enhanced-navigation refreshes. Use `RestoreBehavior.SkipInitialValue` or `SkipLastSnapshot` to suppress restoration during prerendering or reconnection; `RegisterOnRestoring` provides imperative control (10.0).

## TempData and session-backed parameters

- `AddRazorComponents()` registers TempData and provides `ITempData` as a cascading value during server-side rendering. The default cookie provider uses Data Protection; `SessionStorageTempDataProvider` is an alternative. `Get`, `Peek`, and `Keep` control one-time consumption (11.0-preview.2).
- `[SupplyParameterFromTempData]` binds an SSR component property to TempData. Reading consumes the value; assigning writes it back for the next request. Set `Name` when the property and TempData key differ (11.0-preview.4).
- `[SupplyParameterFromSession]` reads and writes SSR component properties through HTTP session, serializes them with `System.Text.Json`, and saves changes before the response. Configure a backing cache, `AddSession`, and `UseSession`; use `Name` for a different session key (11.0-preview.5).

## Identity and passkeys

- Existing Blazor Web Apps can adopt passkey authentication through the ASP.NET Core migration path rather than recreating the app (10.0-migration).
- When upgrading an older Individual Accounts app to `<BlazorDisableThrowNavigationException>true</BlazorDisableThrowNavigationException>`, remove the `InvalidOperationException` thrown by `RedirectTo` and all five `[DoesNotReturn]` attributes from `Components/Account/IdentityRedirectManager.cs` (10.0-migration).
- The Blazor Web App template recognizes authenticator AAGUIDs and suggests friendly names for Google Password Manager, iCloud Keychain, Windows Hello, 1Password, and Bitwarden. Unknown authenticators still use the rename page; extend mappings in `PasskeyAuthenticators.cs` (11.0-preview.2).
