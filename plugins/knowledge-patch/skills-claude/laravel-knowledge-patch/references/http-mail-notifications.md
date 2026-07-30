# HTTP Client, Mail, Notifications, and Broadcasting

## HTTP responses and streaming

- **Custom event streams** `[2025-03]`: `response()->eventStream()` accepts
  custom event names and a start message.
- **Unescaped JSON Unicode** `[13.0-upgrade]`: `Js::from()` applies
  `JSON_UNESCAPED_UNICODE`; exact output assertions now contain characters
  rather than `\u` escapes.

## HTTP client recording, failures, and policy

- **Record real requests** `[2025-03]`: `Http::record()` records traffic without
  faking responses; inspect it with `Http::recorded()`.
- **Retry middleware exceptions** `[2025-04]`: `retry()` now retries exceptions
  thrown by client middleware as well as response and connection failures.
- **Normalized connection exceptions** `[2025-06]`: certificate and connection
  failures use Laravel's HTTP-client exception abstraction instead of leaking
  Guzzle exceptions.
- **Per-request exception truncation** `[2025-06]`: set a pending request's
  `RequestException` message limit with `truncateExceptionsAt()`.
- **Allowed real URLs in tests** `[2025-08]`:
  `preventStrayRequests(allowedUrls: [...])` blocks unexpected traffic while
  permitting matching real endpoints.
- **HTTP response JSON flags** `[2026-01]`: pass flags such as
  `JSON_BIGINT_AS_STRING` to `Response::json(flags: ...)`.
- **PSR client compatibility** `[2026-06]`: Laravel 13's HTTP client can be
  passed directly where a PSR HTTP client is required.
- **HTTP query helpers** `[2026-07]`: use `Http::query()` and test-request
  `query()` or `queryJson()` helpers for query data.

## Pools, batches, promises, and hooks

- **HTTP request batches** `[2025-09]`: coordinate groups of outgoing requests
  with `Http::batch()`.
- **Deferred batches** `[2025-10]`: call `defer()` to schedule a batch instead
  of sending it immediately.
- **Pool and batch concurrency** `[2025-10]`: both coordinators accept a
  concurrency bound.
- **Fluent asynchronous requests** `[2025-12]`: pending-request methods may
  return promises; pools use `FluentPromise`; `Pool` and `Batch` expose
  `newRequest()`.
- **HTTP lifecycle hooks** `[2025-12]`: `withRequestContext()` supplies request
  context, and callbacks can run after response construction.
- **Pool default concurrency** `[13.0.0]`: pools made from `PendingRequest`
  default to concurrency two; set an explicit value when needed.

## Mail transports and rendering

- **Configurable transport retry periods** `[2025-04]`: configure retry periods
  on round-robin and failover mail transports.
- **Raw Resend attachments** `[2025-06]`: send raw, non-encoded attachment
  content through Resend.
- **Inline Resend attachments** `[2025-08]`: Resend preserves inline and
  content-ID attachments.
- **Encoded Markdown strings** `[2025-05]`: `EncodedHtmlString` marks content
  for HTML encoding while `HtmlString` remains raw; Markdown mail can toggle
  this behavior.
- **Custom Markdown extensions** `[2026-03-laravel-12]`: the Markdown mail
  renderer loads application-defined Markdown extensions.
- **Password reset subject** `[13.0-upgrade]`: the default subject is
  `Reset your password`; update exact assertions and translations.
- **Cloudflare delivery** `[2026-04]`: Laravel mail supports Cloudflare Email
  Service.
- **Line-break rejection** `[2026-05]`: mail rejects email addresses containing
  line breaks.
- **Optional mail names** `[2026-07]`: mail configuration entries no longer
  require `name`.
- **SES tenants** `[2026-07]`: the SES v2 transport supports SES tenants.

## Mailables and notifications

- **Database notification timestamps** `[12.0.0]`: set a database notification's
  read time with `readAt()`.
- **Notification failure event** `[2025-04]`: failed channel sends dispatch
  `NotificationFailed`.
- **Custom queued-notification jobs** `[2025-06]`: queued notification dispatch
  can override the default `SendQueuedNotifications` job class.
- **Transport exception on failure** `[2025-07]`: `NotificationFailed` includes
  the originating `TransportException`.
- **Macroable notifications** `[2026-01]`: `Notification` supports macros.
- **Notification lifecycle** `[2026-02]`: `sendNow()` preserves notification
  mutations made in `via()`; define `afterSending()` for post-delivery logic.
- **Delayed mailables** `[2026-02]`: `Mailable::later()` applies its delay to
  the underlying `SendQueuedMailable` job.

## Broadcasting

- **Safe broadcasting installation** `[2026-04]`: `install:broadcasting` invokes
  Yarn with `--ignore-scripts`, so package lifecycle scripts do not run.
