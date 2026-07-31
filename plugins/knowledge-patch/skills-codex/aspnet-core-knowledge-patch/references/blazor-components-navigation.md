# Blazor Components, Navigation, and Circuits

These component, navigation, and circuit behaviors apply to ASP.NET Core
`10.0`.

## Same-page navigation preserves scroll position

`NavigationManager.NavigateTo` no longer scrolls to the top when navigation
stays on the current page, including changes limited to the query string or
fragment. Do not add scroll restoration code based on the earlier behavior.

## `NavLinkMatch.All` compares only the path

`NavLinkMatch.All` ignores query strings and fragments. A link remains active
when its path matches even if either value changes. Set the compatibility
switch before relying on the earlier whole-URI matching behavior:

```csharp
AppContext.SetSwitch(
    "Microsoft.AspNetCore.Components.Routing.NavLink.EnableMatchAllForQueryStringAndFragment",
    true);
```

## Not Found routing

Call `NavigationManager.NotFound()` to set a 404 response during static SSR or
to signal the router during interactive rendering. Use `Router.NotFoundPage`
to choose the routed component and `NavigationManager.OnNotFound` to customize
handling. The old router `<NotFound>` fragment is unsupported.

```razor
<Router AppAssembly="@typeof(Program).Assembly"
        NotFoundPage="typeof(Pages.NotFound)">
    <Found Context="routeData">
        <RouteView RouteData="@routeData" />
    </Found>
</Router>
```

## Reconnection state notifications

The template `ReconnectModal` keeps CSS and JavaScript collocated rather than
injecting styles, so it works with strict CSP `style-src` policies.
Reconnection transitions dispatch the
`components-reconnect-state-changed` browser event. Consumers must recognize
the `retrying` state as part of the state model.

## Server circuit state resumption

A server-side Blazor circuit can retain state across an extended lost
connection or a proactive pause and resume. This can preserve unsaved state,
but a full-page refresh still discards the circuit and its state.

## Direct JavaScript object interop

`IJSRuntime` and `IJSObjectReference` provide
`InvokeConstructorAsync`, `GetValueAsync`, and `SetValueAsync` for constructing
JavaScript objects and reading or writing properties. In-process object
references have synchronous equivalents.

```csharp
var instance = await JSRuntime.InvokeConstructorAsync(
    "jsInterop.TestClass", "Blazor!");
var text = await instance.GetValueAsync<string>("text");
await instance.SetValueAsync("text", "updated");
```
