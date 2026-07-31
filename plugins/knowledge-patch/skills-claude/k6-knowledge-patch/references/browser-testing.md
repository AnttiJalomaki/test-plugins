# Browser Testing

## Locator selection and filtering

### Count matches without a visibility wait (since 1.1.0)

`Locator.count()` returns the number of matches immediately; it does not wait
for those elements to become visible.

```javascript
const inputs = page.locator('input');
expect(await inputs.count()).toEqual(3);
```

### Select a positional match (since 1.1.0)

Use `first()`, `nth(index)`, or `last()` when a locator matches multiple
elements:

```javascript
const paragraphs = page.locator('p');
await expect(await paragraphs.first()).toContainText('QuickPizza');
await expect(await paragraphs.nth(4)).toContainText('QuickPizza Labs.');
await expect(await paragraphs.last()).toContainText('Contribute');
```

### Filter by text (since 1.3.0)

`locator.filter()` accepts `hasText` and `hasNotText`. These options also work
when constructing locators from a page, frame, locator, or `FrameLocator`.

```javascript
const selected = page.locator('li').filter({ hasText: 'Product 2' });
const others = page.locator('li').filter({ hasNotText: /Product 2/ });
```

### Rely on visibility retries (since 1.3.0)

Locator actionability methods retry when the target is not visible. An element
that is temporarily hidden can become actionable instead of causing an
immediate failure.

### Expect DOM order from query-all methods (since 1.4.0)

Browser `QueryAll` methods return their matching elements in DOM order.

## Forms, keyboard input, and page evaluation

### Select options by label (since 1.2.0)

Pass a string label to `locator.selectOption()` to select an option by its
displayed label.

### Evaluate against a locator (since 1.4.0)

`locator.evaluate()` runs a function in the page context with the matching
element and an optional argument. `locator.evaluateHandle()` runs the same way
but returns a `JSHandle`.

```javascript
const text = await page.locator('#pizza-name')
  .evaluate((element, suffix) => element.textContent + suffix, '!');
const handle = await page.locator('#pizza-name')
  .evaluateHandle(element => element);
```

### Send the complete keyboard-event sequence (since 1.5.0)

`locator.pressSequentially()` enters text one character at a time and fires
`keydown`, `keypress`, and `keyup` for each character. Its `delay` option is
useful for autocomplete or per-character validation.

```javascript
const input = page.locator('#search');
await input.pressSequentially('test query', { delay: 100 });
```

Use `fill()` for simple form filling. `type()` types gradually but does not
produce the full per-character event sequence of `pressSequentially()`.

## Frames and iframes

### Enter a frame from an iframe locator (since 1.3.0)

`locator.contentFrame()` returns a `FrameLocator`. Locators created from it
target elements inside the iframe and retain locator auto-retry behavior.

```javascript
const payment = page.locator('iframe[name="payment-form"]').contentFrame();
await payment.locator('input[name="card-number"]').fill('4111111111111111');
await payment.locator('button[type="submit"]').click();
```

### Chain frame locators (since 1.6.0)

`frameLocator()` is available on pages, frames, locators, and frame locators.
Chain it to address nested iframes without switching context manually.

```javascript
const frame = page.frameLocator('#payment-iframe');
await frame.locator('#card-number').fill('4242424242424242');
await frame.frameLocator('#nested-frame').locator('#submit').click();
```

## Request routing

### Intercept before sending (since 1.2.0)

`page.route()` intercepts matching requests before they are sent. Its handler
can abort a request, continue it with changes, or fulfill it with a mocked
response.

```javascript
await page.route('**/api/users', async route => {
  await route.fulfill({
    status: 200,
    contentType: 'application/json',
    body: JSON.stringify([{ id: 1, name: 'Mock User' }]),
  });
});
```

### Remove routes (since 1.4.0)

`page.unroute(url)` removes routes registered with exactly the same URL
matcher. `page.unrouteAll()` removes every registered route.

```javascript
const matcher = /.*\/api\/pizza/;
await page.route(matcher, route => route.continue());
await page.unroute(matcher);
```

## Waiting for network activity and page events

### Wait for a response (since 1.3.0)

`page.waitForResponse()` accepts an exact URL string or a regular expression.
Arm the wait together with its triggering action so the response is not
missed.

```javascript
await Promise.all([
  page.waitForResponse(/\/api\/.*\.json$/),
  page.click('button[data-testid="load-data"]'),
]);
```

### Wait for a request (since 1.4.0)

`page.waitForRequest()` also accepts an exact URL or regular expression. Use
the same concurrent-wait pattern:

```javascript
const [request] = await Promise.all([
  page.waitForRequest(/\/api\/.*\.json$/),
  page.click('button[data-testid="load-data"]'),
]);
```

### Wait for a named page event (since 1.5.0)

`page.waitForEvent()` takes either a predicate directly or an options object
with a predicate and timeout. Create the promise before the action that emits
the event.

```javascript
const responsePromise = page.waitForEvent('response', {
  predicate: response => response.url().includes('/api/data'),
  timeout: 5000,
});
await page.click('button#fetch-data');
const response = await responsePromise;
```

## Request lifecycle and redirects

### Observe unsuccessful and completed requests (since 1.6.0)

Pages emit `requestfailed` for unsuccessful requests and `requestfinished` for
completed requests.

```javascript
page.on('requestfailed', request => console.log('failed', request.url()));
page.on('requestfinished', request => console.log('finished', request.url()));
```

### Receive every redirected request event (since 1.7.0)

Handlers for the `response` and `requestfinished` events cover every request in
a redirect chain.

### Avoid duplicate redirect samples (since 1.8.0)

Browser redirects emit request metrics only for the applicable redirect. They
do not re-emit samples for all previous redirects in the chain, so corrected
results contain no duplicate request samples from that behavior.

## Browser metrics and Cloud diagnostics

### Replace First Input Delay with INP (since 1.3.0)

First Input Delay was scheduled to warn in the 1.4 line and be removed in v2.
Replace `browser_web_vital_fid` thresholds and integrations with Interaction to
Next Paint:

```javascript
export const options = {
  thresholds: {
    browser_web_vital_inp: ['p(95)<200'],
  },
};
```

### Filter Cloud browser failures (since 2.1.0)

Browser API failures in Grafana Cloud Logs carry `module=browser`. Filter on
that field to separate them from other log sources.

## Browser network proxies

### Configure a proxy per browser context (since 2.1.0)

`browser.newContext()` accepts a `proxy` object. `proxy.server` is required;
`proxy.bypass` can exclude destinations.

```javascript
const context = await browser.newContext({
  proxy: {
    server: 'http://proxy.test:8080',
    bypass: 'localhost,127.0.0.1',
  },
});
```
