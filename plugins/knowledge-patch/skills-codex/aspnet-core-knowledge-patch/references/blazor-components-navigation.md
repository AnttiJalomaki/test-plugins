# Blazor Components, Navigation, and Circuits

## Navigation and routing

- `NavigationManager.NavigateTo` preserves scroll position for same-page navigation, including query-string and fragment changes (10.0). Code that needs a top-of-page jump must perform it explicitly.
- `NavLinkMatch.All` compares only the path. Query strings and fragments no longer stop a link from being active. Set the `Microsoft.AspNetCore.Components.Routing.NavLink.EnableMatchAllForQueryStringAndFragment` AppContext switch to `true` for the earlier matching behavior (10.0).
- Use `NavigationManager.NotFound()` to set a 404 during static SSR and signal the router during interactive rendering. Configure `Router.NotFoundPage` for the routed component and use `NavigationManager.OnNotFound` for customization. The old `<NotFound>` router fragment is unsupported (10.0).

```razor
<Router AppAssembly="@typeof(Program).Assembly"
        NotFoundPage="typeof(Pages.NotFound)">
    <Found Context="routeData">
        <RouteView RouteData="@routeData" />
    </Found>
</Router>
```

- Resolve navigation against the current URI with `NavigationOptions.RelativeToCurrentUri = true` or `RelativeToCurrentUri="true"` on `NavLink` (11.0-preview.1). Without it, relative targets continue to resolve against the app base URI.
- In Blazor Web Apps, `<BasePath />` in the document head renders a request-derived base path from `NavigationManager.BaseUri`, falling back to `/`. Standalone WebAssembly apps still require a static `<base href="/" />` (11.0-preview.1).
- `NavigationManager.GetUriWithHash` was renamed to `GetUriWithFragment` (11.0-preview.6).

## Rendering and component metadata

- The environment-aware component introduced as `EnvironmentBoundary` was renamed to `EnvironmentView`. It performs case-insensitive matching against comma-separated `Include` or `Exclude` environment names with MVC environment-tag-helper semantics in server and WebAssembly apps (introduced in 11.0-preview.1; rename in 11.0-preview.6).
- `<Label For="() => model.Name" />` uses `[Display]`, then `[DisplayName]`, then the property name. It can stand separately or wrap its input. In the separate form, built-in input components generate the matching `id` (11.0-preview.1).
- `<DisplayName For="() => product.Price" />` renders the localized display metadata outside a label, with the same `[Display]`, `[DisplayName]`, property-name precedence (11.0-preview.1).
- Dynamically rendered MathML now uses the MathML namespace, so ordinary `<math>` markup works during interactive rendering (11.0-preview.1).
- Implement `IComponentPropertyActivator.GetActivator(Type)` to customize how component `[Inject]` properties are populated, including custom containers and context-aware resolution (11.0-preview.1).
- A component using `@rendermode` may receive `ChildContent` and other non-generic `RenderFragment` parameters across a render-mode boundary. The server prerenders the fragment and the interactive component rehydrates the captured render tree (11.0-preview.5).

## Connections, reconnection, and circuit state

- The template `ReconnectModal` collocates its CSS and JavaScript, avoiding injected styles under strict CSP `style-src`. Reconnection changes dispatch `components-reconnect-state-changed`; handle the additional `retrying` state (10.0).
- Server circuit state can survive an extended disconnect or a proactive pause/resume without discarding unsaved state. A full-page refresh still loses the circuit (10.0).
- Configure the underlying `HttpConnectionDispatcherOptions` for Interactive Server with `ServerComponentsEndpointOptions.ConfigureConnection` (11.0-preview.1).

```csharp
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode(options =>
    {
        options.ConfigureConnection = connection =>
            connection.CloseOnAuthenticationExpiration = true;
    });
```

- `blazor.server.js` and `blazor.webassembly.js` accept the nested `circuit` and `webAssembly` startup option objects used by `blazor.web.js`; their original top-level formats remain supported (11.0-preview.1).
- A server can request a graceful pause with `await circuit.RequestCircuitPauseAsync()`. The result says whether the request reached a connected, initialized circuit. There is no active-circuit registry: capture `Circuit` objects in `CircuitHandler.OnConnectionUpAsync`, remove them in `OnCircuitClosedAsync`, and optionally defer a pause client-side with the `Blazor.start` `onPauseRequested` option (11.0-preview.4).
- Configure browser startup from C# with `WithBrowserOptions` for Server, WebAssembly, and Auto render modes. It covers logging, reconnection, SSR DOM preservation, and WebAssembly environment settings (11.0-preview.6).

```csharp
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode()
    .WithBrowserOptions(options =>
    {
        options.Server.ReconnectionMaxRetries = 10;
        options.Ssr.PreserveDom = true;
    });
```

Preview callers must rename `WithBrowserConfiguration` to `WithBrowserOptions`, replace `DisableDomPreservation` with the oppositely valued `PreserveDom`, and replace millisecond-valued `CircuitInactivityTimeoutMs` with `TimeSpan`-valued `CircuitInactivityTimeout`.

## Virtualization and QuickGrid

- `Virtualize<TItem>` measures and adapts to variable item heights at runtime. Its default `OverscanCount` is 15 rather than 3 to improve average-height estimates; `QuickGrid` continues to use 3 (11.0-preview.3).
- `AnchorMode` compensates when content above the viewport changes height. `Beginning` is the default, `End` follows appends while the user remains at the bottom, `None` disables edge pinning, and the flags can be combined. Supply `ItemComparer` when refreshed reference objects need key-based identity (11.0-preview.4).
- Set `InitialIndex` to choose the first virtualized item and call `ScrollToIndexAsync` after the first interactive render. Calls before that render throw, indexes are clamped, and a later call or user scroll supersedes in-flight scrolling (11.0-preview.6).
- `QuickGrid` sorts and paginates during static SSR through enhanced forms whose state is in the query string. This makes links refreshable and shareable without a circuit or WebAssembly. Disable it with `Microsoft.AspNetCore.Components.QuickGrid.EnableUrlBasedQuickGridNavigationAndSorting`; CSS may need updating because default controls change from `<button>` to `<a>` (11.0-preview.5).
