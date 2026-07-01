---
title: "Looking Behind Playwright's Magic"
date: 2026-07-01
---

In a [previous post](https://josephward.tech/2024-01-21-harmonising-selenium/), I used Chrome performance logs and other approaches to observe CDP network events from Selenium. In Chromium, Playwright listens to those same events, but I didn't look at how it turns them into its own model, or how that work fits around actions like:

```ts
await page.getByRole('button', { name: 'Submit' }).click();
```

Let's break it down.

There are two things going on here. 

Before the input is sent, Playwright checks whether the element can actually receive it. It resolves the locator, waits for relevant element states, scrolls to the target, calculates its coordinates, and checks whether another element would actually receive the input.

Once that input is sent, Playwright may need to track what the browser does next. Our click might begin a navigation, submit a form, open a new page, or trigger more network requests.

My previous post concentrated mainly on that second part. This time I want to follow both through the source.

(Note: Firefox and WebKit have different browser mechanisms behind the same high level Playwright abstractions, so here I am solely talking about Chromium and CDP implementations.)

All the Playwright links are pinned to commit [`ad18048`](https://github.com/microsoft/playwright/commit/ad18048db947ca0a47c7fa59b77718e3a06afafe) from 1 July 2026. 

## It starts with a locator

I began in:

```text
packages/playwright-core/src/client/locator.ts
```

`Locator.click()` doesn't do much by itself. It passes the locator's selector to the frame and enables strict matching:

```ts
async click(options: channels.ElementHandleClickOptions & TimeoutOptions = {}): Promise<void> {
  return await this._frame.click(this._selector, { strict: true, ...options });
}
```

That is the real implementation in [`locator.ts`, lines 1425–1429](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/client/locator.ts#L1425-L1429).

There are two useful things to say about these few lines.

The locator has a selector, not an element fetched earlier. Playwright can therefore resolve it against the document when the action is attempted. That alone helps on pages where a renderer might remove one node and insert another that represents the same thing (leading to a potentially confusing stale element exception).

The `strict: true` option means the action expects one match. If the locator resolves to several buttons, Playwright reports an ambiguity rather than choosing the first.

I like that failure mode. A selector matching several Submit buttons is usually something I want to fix, not something I want the framework to conceal.

The locator call crosses Playwright's client/server boundary and eventually reaches the server-side element action code in:

```text
packages/playwright-core/src/server/dom.ts
```

That is where most of the pre-click "magic" starts to appear.

## A click may need retrying

The internal click path passes the mechanics into Playwright's pointer-action code. The eventual mouse action itself is surrounded by `_retryAction`, which contains a retry loop used for element operations.

One of the first things it defines is the delay schedule:

```ts
// We progressively wait longer between retries, up to 500ms.
const waitTime = [0, 20, 100, 100, 500];
```

The delay schedule is in [`dom.ts`, lines 2754–2774](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/dom.ts#L2754-L2774).

The loop attempts the action again until it succeeds, fails in a non-retryable way, or hits the surrounding timeout. It handles results like an element not being visible or being outside the viewport. The following part of the same method shows those branches in [`dom.ts`, lines 2788–2815](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/dom.ts#L2788-L2815).

We are now into a more useful description of auto-waiting than saying Playwright magically knows when the page is ready.

It does not have or need a universal definition of page readiness. It decides whether a requested action can proceed.

The distinction matters because many modern pages never become completely "inactive" so therefore completely "ready". It may keep a WebSocket open, poll for notifications, or load extra material after the main interface is usable.

Therefore, the narrower question is whether a particular element can receive a particular action.

## What Playwright checks before sending input

For a normal click, Playwright checks states including:

```text
visible
enabled
stable
```

The Playwright-side logic calls `injected.checkElementStates` against the target element. That path can be followed in [`dom.ts`](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/dom.ts).

The three states solve different problems.

A node being present in the DOM does not mean it has useful rendered geometry. A disabled control may be perfectly visible but still should not receive a normal user action. A button may also be visible and enabled while moving across the page due to an animation.

The stable check observes the element across animation frames instead of adding one fixed delay to every action.

(I doubt this makes races impossible. The application can always change again after a check. Playwright is reducing the common timing window rather than claiming to make the browser bulletproof.)

Once those checks pass, Playwright scrolls the target into view. During retries, it can try different alignments instead of relying on one scroll position. The source comments explain that a normal scroll can leave an element underneath an overlay, so an alternative alignment may produce a usable target.

Playwright then calculates a coordinate from the element's geometry.

At this point it has translated something semantically resembling:

```text
the button named Submit
```

into something the browser can definitely use:

```text
a point at x and y
```

It still needs to determine whether the "point" belongs to the intended element.

## What `force` skips

Following the source makes `force: true` less mysterious:

```ts
await locator.click({ force: true });
```

The forced path skips normal actionability work (the visible, enabled and stable checks above, plus the hit-target check described next) and proceeds straight towards the input.

I would read it as:

> Attempt the input without requiring Playwright's usual evidence that a user could perform it.

It does not make the click more realistic. It removes some of the checks designed to establish realism.

There are valid uses for that, but it can also hide the interesting problem: a loading overlay never went away, the locator found a hidden duplicate or the control remained disabled.

The network side is separate. If the forced input triggers a navigation or requests, Playwright can still observe those browser events. The question is whether the test still demonstrates an interaction available to a user.

## Visibility is not hit testing

A visible button can sit underneath another element.

A loading overlay may be nearly transparent, leaving the button visible to a person, while still taking the pointer event. An element can therefore pass a visibility check without being the browser's hit target.

Playwright uses injected code to check the "point" and install a hit-target interceptor:

```ts
const handle = await progress.race(
  this._evaluateHandleInUtility(
    ([injected, node, { actionType, hitPoint, trial }]) =>
      injected.setupHitTargetInterceptor(node, actionType, hitPoint, trial),
    { actionType, hitPoint, trial: !!options.trial } as const
  )
);
```

This is real Playwright code from [`dom.ts`, lines 3015–3039](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/dom.ts#L3015-L3039).

If the browser reports that another element owns the "point", Playwright returns a hit-target description to the outer retry loop. That is the source of messages saying that a particular overlay or container intercepts pointer events.

It is a more specific error, and the hit-target result participates in deciding whether the click should be attempted again.

Frames make this more involved because the coordinate must be checked through the frame order. Playwright has a separate frame hit-target step before setting up the interceptor.

I am leaving most of that alone here.

## Chromium receives a mouse event

When the element checks, scrolling and hit testing succeed, Playwright calls its mouse abstraction with the calculated point.

For Chromium, the final implementation is in:

```text
packages/playwright-core/src/server/chromium/crInput.ts
```

The `RawMouseImpl` sends the CDP command `Input.dispatchMouseEvent`.

A move is sent like this:

```ts
await progress.race(this._client.send('Input.dispatchMouseEvent', {
  type: 'mouseMoved',
  button,
  buttons: toButtonsMask(buttons),
  x,
  y,
  modifiers: toModifiersMask(modifiers),
  force: buttons.size > 0 ? 0.5 : 0,
}));
```

The real implementation, including `mousePressed` and `mouseReleased`, is in [`crInput.ts`, lines 800–893](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/chromium/crInput.ts#L800-L893).

This is not the same as evaluating:

```ts
element.click();
```

inside the page.

The DOM method invokes the element's actual, programmatic click behaviour. The CDP route sends browser input at a position.

That difference is the reason for much of Playwright's action code. Chromium's input domain does not understand a role selector or an accessible name. Playwright has to resolve the semantic target, inspect its current state and produce valid coordinates before Chromium can do anything with it.

The input is only half of the story. Playwright also needs to know what that input causes.

## The network path starts in `crNetworkManager`

The Chromium-specific network implementation is:

```text
packages/playwright-core/src/server/chromium/crNetworkManager.ts
```

When Playwright attaches a CDP session, it registers listeners for the browser's Network and Fetch domain events.

The source includes:

```ts
eventsHelper.addEventListener(session, 'Fetch.requestPaused', ...);
eventsHelper.addEventListener(session, 'Fetch.authRequired', ...);
eventsHelper.addEventListener(session, 'Network.requestWillBeSent', ...);
eventsHelper.addEventListener(session, 'Network.requestWillBeSentExtraInfo', ...);
eventsHelper.addEventListener(session, 'Network.requestServedFromCache', ...);
eventsHelper.addEventListener(session, 'Network.responseReceived', ...);
eventsHelper.addEventListener(session, 'Network.responseReceivedExtraInfo', ...);
eventsHelper.addEventListener(session, 'Network.loadingFinished', ...);
eventsHelper.addEventListener(session, 'Network.loadingFailed', ...);
```

Those registrations are in [`crNetworkManager.ts`, lines 2199–2217](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/chromium/crNetworkManager.ts#L2199-L2217). The same setup also registers WebSocket lifecycle and frame listeners in [`lines 2219–2229`](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/chromium/crNetworkManager.ts#L2219-L2229).

These are very close to the events I accessed through Selenium's Chrome performance log in the previous article.

The difference is not that Playwright has access to a completely different network. It has an implementation that uses those browser events, deals with their ordering, and converts them into its own internal model.

## Predictably, CDP does not deliver one tidy request object

It would be great if Chromium emitted one event containing the complete request, followed by one event containing the complete response. It doesn't.

Request and response information can arrive across different protocol events. When interception is enabled, Playwright may need to correlate `Fetch.requestPaused` with `Network.requestWillBeSent`. Redirects reuse and transform request relationships. Service workers (background scripts that can intercept network requests) introduce cases where some expected events are absent. Extra request and response headers are delivered through separate events.

One awkward case appears in `_onRequestPaused`:

```ts
if (!event.networkId) {
  // Fetch without networkId means that request was not recognized by inspector, and
  // it will never receive Network.requestWillBeSent. Continue the request to not affect it.
  sessionInfo.session._sendMayFail('Fetch.continueRequest', { requestId: event.requestId });
  return;
}
```

See [`crNetworkManager.ts`, lines 2502–2511](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/chromium/crNetworkManager.ts#L2502-L2511).

Playwright cannot wait to correlate that paused request with a `Network.requestWillBeSent` event because the source says that event will never arrive. So it continues the request instead.

There is another special case for service workers. In `_onResponseReceived`, Playwright explains that frame-level requests handled by a service worker may never produce a `requestPaused` event. It may therefore create the request when the response arrives. See [`crNetworkManager.ts`, lines 2865–2888](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/chromium/crNetworkManager.ts#L2865-L2888).

Header information has its own ordering problem. The source comment for `ResponseExtraInfoTracker` says the ordinary request/response events and their `ExtraInfo` events are dispatched on different, unassociated channels and may arrive in random order. The tracker associates them so the extra headers are reliably available by `requestfinished`. See [`crNetworkManager.ts`, lines 3404–3421](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/chromium/crNetworkManager.ts#L3404-L3421).

This is not glamorous work, but it is probably part of the reason the public API feels "magic" vs wrestling with the underlying protocol yourself.

## Turning protocol events into Playwright events

Once Chromium's network events have been linked with a Playwright request, they pass into the page's `FrameManager`.

The request path tracks the request as in flight, associates document requests with pending navigation and fires the public request event:

```ts
requestStarted(request: network.Request, route?: network.RouteDelegate) {
  const frame = request.frame();
  this._inflightRequestStarted(request);
  if (frame && request._documentId)
    frame._setPendingDocument({ documentId: request._documentId, request });
  // ...
  this._page.addNetworkRequest(request);
  this._page.emitOnContext(BrowserContext.Events.Request, request);
  // ...
}
```

That code is in [`frames.ts`, lines 2787–2813](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L2787-L2813).

Responses and completion events are fired nearby:

```ts
requestReceivedResponse(response: network.Response) {
  if (response.request()._isFavicon)
    return;
  this._page.emitOnContext(BrowserContext.Events.Response, response);
}

reportRequestFinished(request: network.Request, response: network.Response | null) {
  this._inflightRequestFinished(request);
  if (request._isFavicon)
    return;
  this._page.emitOnContext(BrowserContext.Events.RequestFinished, { request, response });
}
```

The real implementations, including request failure handling, are in [`frames.ts`, lines 2816–2858](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L2816-L2858).

That is how low-level CDP events become the public interface used by tests:

```ts
page.on('request', request => {
  console.log(request.method(), request.url());
});

page.on('response', response => {
  console.log(response.status(), response.url());
});

page.on('requestfinished', request => {
  console.log('finished', request.url());
});

page.on('requestfailed', request => {
  console.log('failed', request.url());
});
```

## How Playwright implements `networkidle`

The in-flight request handling lives in `frames.ts`.

When a request starts, Playwright adds it to the owning frame's `_inflightRequests` set. If it is the first active request, it stops that frame's network-idle timer.

When a request finishes or fails, Playwright removes it. If the set becomes empty, it starts the timer again:

```ts
private _inflightRequestFinished(request: network.Request) {
  const frame = request.frame();
  if (this._isExcludedFromNetworkIdle(request) || !frame)
    return;
  if (!frame._inflightRequests.has(request))
    return;
  frame._inflightRequests.delete(request);
  if (frame._inflightRequests.size === 0)
    frame._startNetworkIdleTimer();
}

private _inflightRequestStarted(request: network.Request) {
  const frame = request.frame();
  if (this._isExcludedFromNetworkIdle(request) || !frame)
    return;
  frame._inflightRequests.add(request);
  if (frame._inflightRequests.size === 1)
    frame._stopNetworkIdleTimer();
}
```

The source is in [`frames.ts`, lines 2883–2915](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L2883-L2915).

The implementation deliberately excludes favicons and EventSource connections:

```ts
private _isExcludedFromNetworkIdle(request: network.Request): boolean {
  if (request._isFavicon)
    return true;
  if (request.resourceType() === 'eventsource')
    return true;
  return false;
}
```

See [`frames.ts`, lines 2918–2927](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L2918-L2927).

A favicon is browser housekeeping rather than meaningful application activity. An EventSource connection is long-lived by design, so including it would prevent pages using server-sent events from ever reaching network idle.

The idle state is recalculated through the frame tree (the page and its nested iframes). Lifecycle state is recorded on frames, and clearing lifecycle state resets the timer around the remaining current-navigation request. The relevant lifecycle and reset handling can be followed in [`frames.ts`, lines 3176–3212](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L3176-L3212).

It's already more specific than saying "Playwright waits until there are no requests".

A closer explanation is that Playwright tracks in-flight requests per frame (each iframe or the main page counts separately), starts an idle timer when the set becomes empty, and combines that state through the frame hierarchy.

It also explains why `networkidle` is not always a universal sign that an application is ready.

A quiet network says nothing about a CSS animation, a client-side timer, data already queued for rendering or an application bug that prevented the expected request from starting.

## How the click starts waiting before navigation happens

There is another race to solve.

Suppose Playwright sent the click and only then started listening for navigation. A sufficiently fast navigation would begin before a listener was installed.

The pointer action avoids that by running the input inside `waitForSignalsCreatedBy`:

```ts
async waitForSignalsCreatedBy<T>(
  progress: Progress,
  waitAfter: boolean,
  action: (progress: Progress) => Promise<T>
): Promise<T> {
  if (!waitAfter)
    return action(progress);
  const barrier = new SignalBarrier(progress);
  this._signalBarriers.add(barrier);
  try {
    const result = await action(progress);
    await progress.race(this._page.delegate.inputActionEpilogue());
    await barrier.waitFor(progress);
    // Resolve in the next task, after all waitForNavigations.
    await new Promise<void>(makeWaitForNextTask());
    return result;
  } finally {
    this._signalBarriers.delete(barrier);
  }
}
```

That is the real method in [`frames.ts`, lines 2547–2574](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L2547-L2574).

The `SignalBarrier` acts as a gate: it holds the action open until any navigation the click started has been acknowledged.

When a frame may request navigation, active gates are retained. When the possible request has been resolved, they are released. If a frame actually requests a navigation, Playwright adds that frame navigation to the gate. See [`frames.ts`, lines 2578–2615](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L2578-L2615).

The gate is not waiting for every network request caused by a click.

The gate mainly coordinates browser signals, particularly navigation, created by the action. General network observation continues through the network manager and public request events.

A click may produce an API request without navigating. Playwright cannot safely infer which arbitrary response represents the business completion the test cares about. A test can express that relationship explicitly:

```ts
const responsePromise = page.waitForResponse(
  response =>
    response.url().endsWith('/orders') &&
    response.request().method() === 'POST'
);

await page.getByRole('button', { name: 'Submit' }).click();

const response = await responsePromise;
```

The response wait is created before the click for the same general reason as the internal gate: it prevents a fast event from being missed. This is why the Playwright documentation tells you to set up `waitForResponse` before the action that triggers it. The test is applying the same principle that Playwright uses internally with `waitForSignalsCreatedBy`. Install the listener first, then perform the action.

## Connecting this back to Selenium

Following the source changes how I would describe my previous experiment, but it does not make it wrong.

The Chrome performance log approach was observing real CDP network events. Playwright's Chromium network manager listens to the same family of events.

What Playwright adds is a substantial layer around them. It installs protocol listeners, correlates Fetch and Network-domain events, handles redirects and service workers, constructs request and response objects, exposes them as public events, tracks requests per frame, calculates network idle, and links page-loading requests to the navigations that caused them.

My Selenium examples covered a narrower problem: determine whether network activity appears to be in progress or not.

That can still be enough for a particular test suite. The heavier implementation in Playwright exists because a reusable framework has to handle more varied browser behaviour, support different test types and writing styles, and build in redundancy that an application-specific helper does not need.

A network counter also does not address the other half of the problem.

Before the request or navigation exists, Playwright still has to make the click happen. That involves locator resolution, element states, scrolling, geometry, hit testing and browser input.

Network awareness cannot tell us that a transparent overlay is blocking the click. Pre-click checks cannot tell us that the expected API response returned correct data.

They solve different parts of synchronisation.

## Wrapping up

I started with one line:

```ts
await page.getByRole('button', { name: 'Submit' }).click();
```

Following it through the source produced two connected systems.

The first prepares and sends the input:

```text
resolve the locator
check visible, enabled and stable
scroll the element
calculate a point
check the hit target
dispatch browser input
```

The second watches what the browser does:

```text
receive raw request and response events from the browser
normalise them into Playwright's own objects
track which requests are still in progress
fire public events that tests can listen to
link page-loading requests to navigations
wait for navigations triggered by the action
calculate network-idle state across the page and its iframes
```

These are the browser and network signals I was trying to expose to Selenium in the previous article. Playwright really does use them.

It also does considerably more than count requests.

The Chromium protocol is noisy and occasionally incomplete. Related information arrives through different events and can arrive in an inconvenient order. Requests can redirect, come from service workers, be served from cache or fail before all expected events appear. Playwright contains code for turning that into a more stable model.

None of this means that a Playwright test should wait for `networkidle` after every action. The implementation shows why it is only one signal. It measures a quiet period where no tracked requests are active. It does not measure whether the application is correct or useful.

I still do not think the conclusion is simply to choose Playwright over Selenium.

Selenium users can observe CDP events, execute browser-side JavaScript and build application-specific waits. Playwright has chosen to make more of that coordination part of the framework itself.

The useful lesson is to be clear about what we are waiting for.

Before a click, that might be a stable and unobstructed target. After it, that might be a navigation lifecycle event (like `load` or `DOMContentLoaded`) or one specific API response. Sometimes network idle is useful. Sometimes it is the wrong question entirely.

I may still try reproducing Playwright's "is something blocking the click" check in Selenium.

I will not be reproducing `crNetworkManager`. Life is short.

Did I get something wrong? Have you been down a similar rabbit hole in Playwright or Selenium? Please let me know by getting in touch at [joseph@josephward.tech](mailto:joseph@josephward.tech).
