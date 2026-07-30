# Runtime APIs and JavaScript Behavior

This reference organizes runtime and client behavior from batch `1.8.x`.

## Limit Channels Per Transport

Phoenix 1.8.9 adds `max_channels_per_transport`, which defaults to 100. It
bounds how many channel processes a single client can create over one
transport. Applications that intentionally multiplex more than 100 channels
must raise the option explicitly; treat the change as a resource and abuse
boundary rather than disabling it casually.

## Assign in Bulk or from Existing Assigns

`Phoenix.Socket.assign/2` accepts a function of the current assigns. The map
returned by the function is merged into the socket assigns.

```elixir
socket = Phoenix.Socket.assign(socket, fn assigns ->
  %{count: assigns.count + 1}
end)
```

`Phoenix.Controller.assign/2` also accepts the functional form, plus maps and
keyword lists, matching the bulk-assignment style available in LiveView.

```elixir
conn = Phoenix.Controller.assign(conn, current_user: user, locale: "en")
```

## Guard Channel Test Assertions

Since Phoenix 1.8.4, `assert_push`, `assert_broadcast`, and `assert_reply`
support guards. Constrain a received payload directly instead of following the
receive assertion with a separate shape assertion.

```elixir
assert_push "updated", payload when is_map(payload)
```

## JavaScript Socket Lifecycle

The JavaScript socket stops reconnect attempts while the page is hidden. Tests
and monitoring that expect continuous retry traffic should account for page
visibility before diagnosing a stalled reconnection loop.

LongPoll can use `fetch()` when `XMLHttpRequest` is unavailable. This is a
client transport fallback; LongPoll still has to be enabled explicitly on the
server.

## Presence Behavior

Phoenix 1.8.9 prevents Presence keys matching members of `Object.prototype`
from crashing the JavaScript client. Do not add application-side bans on those
keys merely to work around the earlier crash when the client has the fix.

Presence also accepts a custom dispatcher for `presence_diff` broadcasts. Use
the dispatcher when the application needs to control how Presence diffs are
scheduled or delivered rather than replacing Presence's synchronization logic.

