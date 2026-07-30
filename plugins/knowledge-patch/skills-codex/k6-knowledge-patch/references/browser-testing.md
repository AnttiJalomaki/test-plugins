# Browser testing

## Network observation and lifecycle

Since 1.0.0-rc1, pages emit `request` and `response`:

```javascript
page.on('request', request => console.log(request.url()));
page.on('response', response => console.log(response.url()));
```

Since 1.6.0, `requestfailed` reports unsuccessful requests and
`requestfinished` reports completed requests. Since 1.7.0, `response` and
`requestfinished` handlers receive every request in a redirect chain.

Since 2.1.0, browser API failures delivered to Grafana Cloud Logs carry
`module=browser`, enabling browser-only filters.

## Waiting without races

`page.waitForResponse()` accepts an exact URL or regular expression since
1.3.0. `page.waitForRequest()` supports the same matcher forms since 1.4.0.
Start either wait together with the triggering action:

```javascript
const [response] = await Promise.all([
  page.waitForResponse(/\/api\/.*\.json$/),
  page.click('#load-data'),
]);
```

Since 1.5.0, `page.waitForEvent()` accepts either a predicate directly or an
options object with a predicate and timeout. Arm the promise before the action:

```javascript
const responsePromise = page.waitForEvent('response', {
  predicate: response => response.url().includes('/api/data'),
  timeout: 5000,
});
await page.click('#fetch-data');
const response = await responsePromise;
```

## Request interception

Since 1.2.0, `page.route()` intercepts a request before sending it. A handler
can abort it, continue it with changes, or fulfill it with a mock:

```javascript
await page.route('**/api/users', route => route.fulfill({
  status: 200,
  contentType: 'application/json',
  body: JSON.stringify([{ id: 1, name: 'Mock User' }]),
}));
```

Since 1.4.0, `page.unroute(url)` removes routes registered with exactly the
same matcher and `page.unrouteAll()` removes all routes:

```javascript
const matcher = /.*\/api\/pizza/;
await page.route(matcher, route => route.continue());
await page.unroute(matcher);
```

## Locator sets and filtering

Since 1.1.0:

- `Locator.count()` returns the current match count without waiting for
  visibility.
- `first()`, `nth(index)`, and `last()` select positions from a multi-match
  locator.

Since 1.3.0, `locator.filter()` accepts `hasText` and `hasNotText`. The same
options work when creating locators from a page, frame, locator, or
`FrameLocator`:

```javascript
const product = page.locator('li').filter({ hasText: 'Product 2' });
const others = page.locator('li').filter({ hasNotText: /Product 2/ });
```

Locator actionability APIs also retry when their target is temporarily hidden
since 1.3.0. Browser `QueryAll` results are in DOM order since 1.4.0.

## Frames

Since 1.3.0, `locator.contentFrame()` returns a `FrameLocator` while retaining
locator auto-retry:

```javascript
const frame = page.locator('iframe[name="payment"]').contentFrame();
await frame.locator('input[name="card-number"]').fill('4111111111111111');
```

Since 1.6.0, `frameLocator()` is available on pages, frames, locators, and
frame locators. Chain it for nested iframes:

```javascript
const frame = page.frameLocator('#payment-iframe');
await frame.frameLocator('#nested-frame').locator('#submit').click();
```

## Input, selection, and evaluation

- Since 1.2.0, `locator.selectOption()` accepts string labels.
- Since 1.4.0, `locator.evaluate()` runs page-context code with the element and
  an optional argument; `evaluateHandle()` returns a `JSHandle`.
- Since 1.5.0, `locator.pressSequentially()` enters text character by
  character and fires `keydown`, `keypress`, and `keyup` for each one. Its
  `delay` supports autocomplete and per-character validation. `fill()` is
  simple filling; `type()` is gradual input without the full event sequence.

```javascript
await page.locator('#search').pressSequentially('test query', { delay: 100 });
```

## Browser metrics and redirects

In 1.3.0, First Input Delay was scheduled to warn in the 1.4.x line and be
removed in 2.0.0. Replace `browser_web_vital_fid` thresholds and integrations
with Interaction to Next Paint:

```javascript
export const options = {
  thresholds: { browser_web_vital_inp: ['p(95)<200'] },
};
```

Since 1.8.0, redirects emit request metrics only for the applicable redirect,
not duplicate samples for every earlier redirect in the chain.

## Per-context proxy

Since 2.1.0, `browser.newContext()` accepts a `proxy`. `proxy.server` is
required; `proxy.bypass` excludes destinations:

```javascript
const context = await browser.newContext({
  proxy: {
    server: 'http://proxy.test:8080',
    bypass: 'localhost,127.0.0.1',
  },
});
```
