# Components, Navigation, and Interop

Use this reference for component routing, browser interop, reconnection, server circuits, and
standalone WebAssembly globalization. The stable behavior described here is from batch `10.0`.

## Navigation behavior

### Preserve scroll position for same-page navigation

`NavigationManager.NavigateTo` no longer scrolls to the top when navigation remains on the same
page and changes only its query string or fragment. Do not add scroll restoration that assumes
the earlier reset. If the product requires a reset, implement it explicitly.

### Match `NavLinkMatch.All` by path

`NavLinkMatch.All` ignores the query string and fragment. A link remains active while either
portion changes, provided the path still matches.

Set the following AppContext switch to `true` only when an upgraded app must temporarily retain
the earlier query-and-fragment-sensitive behavior:

```text
Microsoft.AspNetCore.Components.Routing.NavLink.EnableMatchAllForQueryStringAndFragment
```

### Route Not Found responses

Use `NavigationManager.NotFound()` to produce a 404 during static server-side rendering and to
signal the router during interactive rendering. Select the routed page with
`Router.NotFoundPage`, and subscribe to `NavigationManager.OnNotFound` for customization.

```razor
<Router AppAssembly="@typeof(Program).Assembly"
        NotFoundPage="typeof(Pages.NotFound)">
    <Found Context="routeData">
        <RouteView RouteData="@routeData" />
    </Found>
</Router>
```

Do not use the old `<NotFound>` router fragment; it is unsupported in ASP.NET Core 10.

## Reconnection and circuit state

### Observe reconnection states

The current `ReconnectModal` template collocates its CSS and JavaScript instead of injecting
styles. This works with strict CSP `style-src` policies. Reconnection changes dispatch the
`components-reconnect-state-changed` browser event, and the state set includes `retrying`.

Use the event when surrounding UI or telemetry must react to reconnection. Keep custom modal
styles collocated so a strict CSP does not require inline-style exceptions.

### Resume server circuit state

Server-side Blazor can retain circuit state through an extended lost connection or a proactive
pause and resume. This preserves unsaved state, but a full-page browser refresh still discards
the circuit. Design recovery UI so it distinguishes resumable disconnects from reloads.

## Direct JavaScript object interop

`IJSRuntime` and `IJSObjectReference` support JavaScript construction and property access through
`InvokeConstructorAsync`, `GetValueAsync`, and `SetValueAsync`.

```csharp
var instance = await JSRuntime.InvokeConstructorAsync(
    "jsInterop.TestClass",
    "Blazor!");

var text = await instance.GetValueAsync<string>("text");
await instance.SetValueAsync("text", "updated");
```

In-process references expose synchronous counterparts. Prefer asynchronous calls unless the
component is deliberately tied to an in-process runtime.

## Standalone WebAssembly UI culture

Standalone Blazor WebAssembly apps load globalization resources for
`CultureInfo.DefaultThreadCurrentUICulture` as well as
`CultureInfo.DefaultThreadCurrentCulture`. Account for UI-culture resources when estimating the
published payload and when testing localization defaults.
