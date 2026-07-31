# Browser Testing

## Module stability and scenario setup

`k6/browser` is a stable, production-ready module (1.0.0). Configure a browser
scenario and create pages through the module:

```javascript
import { browser } from 'k6/browser';

export const options = {
  scenarios: {
    ui: {
      executor: 'shared-iterations',
      options: { browser: { type: 'chromium' } },
    },
  },
};
```

## Selecting locator matches

### Count without visibility waiting

`Locator.count()` returns the number of matching elements immediately; it does
not wait for them to become visible (since 1.1.0):

```javascript
const inputs = page.locator('input');
expect(await inputs.count()).toEqual(3);
```

### Select by position

Use `first()`, `nth(index)`, and `last()` to select one result from a locator
that matches multiple elements (since 1.1.0):

```javascript
const paragraphs = page.locator('p');
await expect(await paragraphs.first()).toContainText('QuickPizza');
await expect(await paragraphs.nth(4)).toContainText('QuickPizza Labs.');
await expect(await paragraphs.last()).toContainText('Contribute');
```

### Filter by text and retry hidden targets

`locator.filter()` accepts `hasText` and `hasNotText` (since 1.3.0). The same
options work when creating locators from a page, frame, locator, or
`FrameLocator`:

```javascript
const product = page.locator('li').filter({ hasText: 'Product 2' });
const others = page.locator('li').filter({ hasNotText: /Product 2/ });
```

Locator actionability APIs retry when the target is not visible (1.3.0), so a
temporarily hidden element can become actionable instead of failing
immediately.

### Select options by label

`locator.selectOption()` accepts string labels directly (since 1.2.0). Pass the
visible label where label-based selection is more stable than an option value.

### Evaluate in the page context

`locator.evaluate()` executes with the matched element and an optional
argument; `evaluateHandle()` returns a `JSHandle` (since 1.4.0):

```javascript
const text = await page.locator('#pizza-name')
  .evaluate((element, suffix) => element.textContent + suffix, '!');
const handle = await page.locator('#pizza-name')
  .evaluateHandle(element => element);
```

Browser `QueryAll` methods return results in DOM order (1.4.0). Code selecting
from those arrays can rely on document ordering.

## Frames and nested documents

### Convert an iframe locator

`locator.contentFrame()` returns a `FrameLocator` while retaining locator
auto-retry behavior (since 1.3.0):

```javascript
const payment = page.locator('iframe[name="payment-form"]').contentFrame();
await payment.locator('input[name="card-number"]').fill('4111111111111111');
await payment.locator('button[type="submit"]').click();
```

### Chain frame locators

`frameLocator()` is available on pages, frames, locators, and frame locators
(since 1.6.0). Chain it for nested iframes without manually switching context:

```javascript
const frame = page.frameLocator('#payment-iframe');
await frame.locator('#card-number').fill('4242424242424242');
await frame.frameLocator('#nested-frame').locator('#submit').click();
```

## Request interception and routing

`page.route()` intercepts requests before they are sent. Its handler can abort,
continue with changes, or fulfill with a mock response (since 1.2.0):

```javascript
await page.route('**/api/users', async route => {
  await route.fulfill({
    status: 200,
    contentType: 'application/json',
    body: JSON.stringify([{ id: 1, name: 'Mock User' }]),
  });
});
```

Remove one registration with `page.unroute(url)` using exactly the same URL
matcher used for `route()`; use `page.unrouteAll()` to remove every route
(since 1.4.0):

```javascript
const matcher = /.*\/api\/pizza/;
await page.route(matcher, route => route.continue());
await page.unroute(matcher);
```

## Waiting without races

Start a wait before, or concurrently with, the action that emits the matching
traffic.

### Wait for a response

`page.waitForResponse()` accepts an exact URL string or a regular expression
(since 1.3.0):

```javascript
await Promise.all([
  page.waitForResponse(/\/api\/.*\.json$/),
  page.click('button[data-testid="load-data"]'),
]);
```

### Wait for a request

`page.waitForRequest()` accepts an exact URL string or regular expression
(since 1.4.0):

```javascript
const [request] = await Promise.all([
  page.waitForRequest(/\/api\/.*\.json$/),
  page.click('button[data-testid="load-data"]'),
]);
```

### Wait for a named page event

`page.waitForEvent()` accepts either a predicate directly or an options object
with `predicate` and `timeout` (since 1.5.0):

```javascript
const responsePromise = page.waitForEvent('response', {
  predicate: response => response.url().includes('/api/data'),
  timeout: 5000,
});
await page.click('button#fetch-data');
const response = await responsePromise;
```

## Observing request lifecycles

Listen for unsuccessful and completed requests with `requestfailed` and
`requestfinished` (since 1.6.0):

```javascript
page.on('requestfailed', request => console.log('failed', request.url()));
page.on('requestfinished', request => console.log('finished', request.url()));
```

From 1.7.0, `response` and `requestfinished` handlers cover every request in a
redirect chain. From 1.8.0, redirects emit request metrics only for the
applicable redirect rather than re-emitting samples for all earlier redirects.
When upgrading, remove workarounds for missing redirect events or duplicated
redirect metrics.

## Typing-sensitive controls

`locator.pressSequentially()` types one character at a time and fires
`keydown`, `keypress`, and `keyup` for every character (since 1.5.0):

```javascript
const input = page.locator('#search');
await input.pressSequentially('test query', { delay: 100 });
```

Use it for autocomplete or per-character validation. `fill()` performs simple
form filling, while `type()` types gradually without the full per-character
keyboard-event sequence.

## Per-context proxies

`browser.newContext()` accepts a `proxy` option (since 2.1.0). `proxy.server`
is required, and `proxy.bypass` excludes destinations:

```javascript
const context = await browser.newContext({
  proxy: {
    server: 'http://proxy.test:8080',
    bypass: 'localhost,127.0.0.1',
  },
});
```

## Browser measurements and Cloud diagnostics

First Input Delay is being removed. Replace `browser_web_vital_fid`
thresholds and integrations with Interaction to Next Paint (1.3.0):

```javascript
export const options = {
  thresholds: {
    browser_web_vital_inp: ['p(95)<200'],
  },
};
```

Browser API failures in Grafana Cloud Logs carry `module=browser` (2.1.0), so
filter on that field to isolate browser failures from other log sources.
