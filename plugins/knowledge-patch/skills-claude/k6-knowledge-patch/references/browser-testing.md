# Browser Testing

## Observe page network traffic

Pages gained request and response events in `1.0.0-rc1`:

```javascript
page.on('request', request => console.log(request.url()));
page.on('response', response => console.log(response.url()));
```

`requestfailed` and `requestfinished` were added in `1.6.0` for unsuccessful
and completed requests:

```javascript
page.on('requestfailed', request => console.log('failed', request.url()));
page.on('requestfinished', request => console.log('finished', request.url()));
```

As of `1.7.0`, `response` and `requestfinished` handlers observe every request
in a redirect chain.

## Wait without racing the page

`page.waitForResponse()` accepts an exact URL string or regular expression as
of `1.3.0`. Arm it together with the triggering action:

```javascript
await Promise.all([
  page.waitForResponse(/\/api\/.*\.json$/),
  page.click('button[data-testid="load-data"]'),
]);
```

`page.waitForRequest()` added the equivalent request-side behavior in `1.4.0`:

```javascript
const [request] = await Promise.all([
  page.waitForRequest(/\/api\/.*\.json$/),
  page.click('button[data-testid="load-data"]'),
]);
```

`page.waitForEvent()` was added in `1.5.0`. It accepts a predicate directly or
an object with `predicate` and `timeout`; create the promise before the action:

```javascript
const responsePromise = page.waitForEvent('response', {
  predicate: response => response.url().includes('/api/data'),
  timeout: 5000,
});
await page.click('button#fetch-data');
const response = await responsePromise;
```

## Route and mock requests

`page.route()` began intercepting matching requests before transmission in
`1.2.0`. A route handler can abort, continue with changes, or fulfill with a
mock:

```javascript
await page.route('**/api/users', async route => {
  await route.fulfill({
    status: 200,
    contentType: 'application/json',
    body: JSON.stringify([{ id: 1, name: 'Mock User' }]),
  });
});
```

In `1.4.0`, `page.unroute(url)` removes routes registered with exactly the
same URL matcher, while `page.unrouteAll()` removes all routes:

```javascript
const matcher = /.*\/api\/pizza/;
await page.route(matcher, route => route.continue());
await page.unroute(matcher);
```

## Select locator matches

`Locator.count()` returns the match count immediately without waiting for
visibility as of `1.1.0`.

`Locator.first()`, `Locator.nth(index)`, and `Locator.last()` were also added
in `1.1.0`:

```javascript
const paragraphs = page.locator('p');
await expect(await paragraphs.first()).toContainText('QuickPizza');
await expect(await paragraphs.nth(4)).toContainText('QuickPizza Labs.');
await expect(await paragraphs.last()).toContainText('Contribute');
```

`locator.filter()` gained `hasText` and `hasNotText` in `1.3.0`. These filters
also work when creating locators from a page, frame, locator, or
`FrameLocator`:

```javascript
const product = page.locator('li').filter({ hasText: 'Product 2' });
const others = page.locator('li').filter({ hasNotText: /Product 2/ });
```

Locator actionability APIs retry temporarily invisible targets as of `1.3.0`
instead of failing immediately. Browser `QueryAll` methods return results in
DOM order as of `1.4.0`.

## Work with frames

`locator.contentFrame()` returns a `FrameLocator` as of `1.3.0`, preserving
locator auto-retry behavior inside an iframe:

```javascript
const payment = page.locator('iframe[name="payment-form"]').contentFrame();
await payment.locator('input[name="card-number"]').fill('4111111111111111');
```

`frameLocator()` became available on pages, frames, locators, and frame
locators in `1.6.0`, and can be chained for nested frames:

```javascript
const frame = page.frameLocator('#payment-iframe');
await frame.locator('#card-number').fill('4242424242424242');
await frame.frameLocator('#nested-frame').locator('#submit').click();
```

## Evaluate and enter values

`locator.selectOption()` accepts string labels as of `1.2.0`.

`locator.evaluate()` runs code in the page context with the matching element
and an optional argument as of `1.4.0`; `evaluateHandle()` returns a
`JSHandle`:

```javascript
const text = await page.locator('#pizza-name')
  .evaluate((element, suffix) => element.textContent + suffix, '!');
const handle = await page.locator('#pizza-name')
  .evaluateHandle(element => element);
```

`locator.pressSequentially()` was added in `1.5.0`. It types one character at
a time and emits `keydown`, `keypress`, and `keyup` for each character:

```javascript
await page.locator('#search')
  .pressSequentially('test query', { delay: 100 });
```

Use it for autocomplete and per-character validation. `fill()` performs
simple form filling, while `type()` types gradually without the full keyboard
event sequence.

## Browser metrics and redirect corrections

First Input Delay was on a removal path in `1.3.0`. Replace
`browser_web_vital_fid` with Interaction to Next Paint:

```javascript
export const options = {
  thresholds: {
    browser_web_vital_inp: ['p(95)<200'],
  },
};
```

In `1.8.0`, browser redirects stopped re-emitting request metrics for all
earlier redirects in a chain. Each redirect now emits only its applicable
request metrics, eliminating duplicate samples.

## Per-context proxies

`browser.newContext()` accepts a `proxy` option as of `2.1.0`. `proxy.server`
is required; `proxy.bypass` excludes destinations:

```javascript
const context = await browser.newContext({
  proxy: {
    server: 'http://proxy.test:8080',
    bypass: 'localhost,127.0.0.1',
  },
});
```

Browser API failures sent to Grafana Cloud Logs carry `module=browser` as of
`2.1.0`, so browser failures can be filtered from other log sources.

