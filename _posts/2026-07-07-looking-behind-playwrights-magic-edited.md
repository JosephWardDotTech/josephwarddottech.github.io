---
layout: post
title: "Looking Behind Playwright's Magic"
date: 2026-07-07
sitemap: false
---

A Playwright click looks simple:

```ts
await page.getByRole('button', { name: 'Submit' }).click();
```

Chromium, however, cannot click “the button named Submit”. It can only receive input at an x,y position on the page. Before Playwright can send that action, it resolves the locator, determines whether the element can be interacted with, scrolls it into view, calculates its coordinates, and checks that another element will not receive the click instead.

Even so, sending the action is only half the battle. The click may start a navigation, submit a form, open a new page, or trigger network requests. Playwright then needs to coordinate with what the browser does next.

In a [previous post](https://josephward.tech/2024-01-21-harmonising-selenium/), I used several approaches to observe Chrome DevTools Protocol (CDP) network events from Selenium. This time, I followed Playwright's source to see how it turns those low-level browser signals into the higher-level behaviour behind a click.

There are two paths to examine:

1. how Playwright prepares and completes the action
2. how it observes and waits for the consequences.

This article follows the Chromium implementation, where Playwright talks to the browser using CDP. Firefox and WebKit provide the same high-level Playwright abstractions through different underlying code.

All Playwright source links are pinned to commit [`ad18048`](https://github.com/microsoft/playwright/commit/ad18048db947ca0a47c7fa59b77718e3a06afafe) from 1 July 2026.

## It starts with a locator

The trail begins in:

```text
packages/playwright-core/src/client/locator.ts
```

`Locator.click()` does not perform the click itself. It passes the locator's selector to the frame and enables strict matching:

```ts
async click(options: channels.ElementHandleClickOptions & TimeoutOptions = {}): Promise<void> {
  return await this._frame.click(this._selector, { strict: true, ...options });
}
```

That is the implementation in [`locator.ts`, lines 113–115](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/client/locator.ts#L113-L115).

Those few lines reveal two useful design choices.

First, the locator holds a selector rather than an element fetched earlier. Playwright can resolve it against the current document when the action is attempted. That matters on pages where a rendering framework may remove one node and insert another that represents the same thing. Resolving at action time avoids treating an old element reference as though it were still current.

Second, `strict: true` means the action expects exactly one match. If the locator resolves to several buttons, Playwright reports an ambiguity instead of silently choosing the first.

I prefer that failure mode. A selector matching several `Submit` buttons is usually a problem in the test or the interface, not something I want the framework to conceal.

The call then crosses Playwright's client/server boundary and eventually reaches the server-side element action code in:

```text
packages/playwright-core/src/server/dom.ts
```

This is where most of the pre-click “magic” becomes explicit logic.

## A click may need retrying

The internal click path passes the work into Playwright's element interaction code. The eventual mouse action is wrapped by `_retryAction`, which contains a retry loop shared by element operations.

One of the first things it defines is a delay schedule:

```ts
// We progressively wait longer between retries, up to 500ms.
const waitTime = [0, 20, 100, 100, 500];
```

The schedule appears in [`dom.ts`, lines 318–319](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/dom.ts#L318-L319).

The loop attempts the action until it succeeds, encounters a non-retryable failure, or reaches the surrounding timeout. It handles results such as an element not being visible or lying outside the viewport. The branches are shown in [`dom.ts`, lines 297–314](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/dom.ts#L297-L314).

This gives us a more useful description of auto-waiting than saying Playwright somehow knows when a page is “ready”. It does not need a universal definition of readiness. Instead, it asks a narrower and more understandable question: can this requested action proceed now?

That distinction matters because many modern pages never become completely inactive. A page may keep a WebSocket open, poll for notifications, or continue loading secondary content after the main interface is usable. Waiting for the entire page to become abstractly “ready” would be both vague and, in some applications, impossible.

For a click, the practical question is whether this particular target can receive this particular action.

## What Playwright checks before sending input

For a normal click, Playwright checks several element states, including:

```text
visible
enabled
stable
```

The Playwright-side logic calls `injected.checkElementStates` against the target. That path can be followed in [`dom.ts`, lines 432–440](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/dom.ts#L432-L440).

Each state addresses a different failure mode.

A node being present in the DOM does not mean it has useful geometry. A disabled control may be visible but should not receive a normal user action. A button may be visible and enabled while still moving across the page because of an animation.

The stability check therefore observes the element across animation frames rather than adding the same fixed delay to every action.

This does not make races impossible. The application can always change after a check. Playwright is just reducing a common timing window, not making the browser immune to change.

Once those checks pass, Playwright scrolls the target into view. During retries, it can use different alignments rather than relying on one scroll position. The source comments explain why: a normal scroll may leave the element beneath an overlay, while another alignment may expose a usable point.

Playwright then calculates a x,y coordinate from the element's geometry.

At this stage it has translated something semantic:

```text
the button named Submit
```

into something the browser's input system can use:

```text
a point at x and y
```

It still has to establish that the point belongs to the intended element at the moment of interaction.

## What `force` skips

Following this path also makes `force: true` less mysterious:

```ts
await locator.click({ force: true });
```

The forced path skips Playwright's normal actionability logic : the visible, enabled, and stable checks above, along with the hit-target check described next. It proceeds towards dispatching the input without requiring the usual evidence that a user could perform the action.

I would read it as:

> Attempt the action without Playwright first establishing user actionability.

It does not make the click more realistic. In fact, it arguably removes checks intended to establish realism.

There are legitimate uses for this, but it can also hide the interesting problem. Perhaps a loading overlay never disappeared, the locator matched a hidden duplicate, or the control remained disabled.

The network side remains separate. If forced input triggers a navigation or network requests, Playwright can still observe those browser events. The question is whether the test still demonstrates an interaction available to a user.

## Visibility is not targetabity 

A visible element is not necessarily the element that would receive an action.

A loading overlay might be nearly transparent, leaving the button visible to a person while still intercepting the click. An element can therefore pass a visibility check without being the browser's target.

Playwright uses injected code to check which element would receive the pointer event and to install a hit-target interceptor:

```ts
const handle = await progress.race(
  this._evaluateHandleInUtility(
    ([injected, node, { actionType, hitPoint, trial }]) =>
      injected.setupHitTargetInterceptor(node, actionType, hitPoint, trial),
    { actionType, hitPoint, trial: !!options.trial } as const
  )
);
```

This comes from [`dom.ts`, line 470](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/dom.ts#L470).

If another element owns the 'point', Playwright returns a hit-target description to the outer retry loop. That is the source of errors reporting that a particular overlay or container intercepts pointer events.

The result feeds back into the retry logic, which can decide whether to attempt the click again.

Frames make the calculation more involved because the point must be checked through the frame hierarchy. Playwright performs a separate frame hit-target step before installing the interceptor, but I will leave most of that path unexplored here.

## Chromium receives a mouse event

Once the element checks, scrolling, and targetabity succeed, Playwright calls its mouse abstraction with the calculated point.

For Chromium, the final implementation is in:

```text
packages/playwright-core/src/server/chromium/crInput.ts
```

`RawMouseImpl` sends the CDP command `Input.dispatchMouseEvent`.

A mouse move is dispatched like this:

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

The implementation, including `mousePressed` and `mouseReleased`, is in [`crInput.ts`, lines 104–164](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/chromium/crInput.ts#L104-L164).

This is not the same as evaluating the following inside the page:

```ts
element.click();
```

The DOM method invokes the element's programmatic click behaviour. The CDP route sends browser input at a position.

That difference explains much of Playwright's action code. Chromium's input domain does not understand a 'role selector' or an 'accessible name'. Playwright must resolve the semantic target, inspect its current state, and produce valid coordinates before Chromium can act on it.

Now for the other side of the story...

## The network path starts in `crNetworkManager`

The Chromium-specific network implementation is in:

```text
packages/playwright-core/src/server/chromium/crNetworkManager.ts
```

When Playwright attaches a CDP session, it registers listeners for events from the browser's Network and Fetch domains.

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

Those registrations appear in [`crNetworkManager.ts`, lines 67–75](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/chromium/crNetworkManager.ts#L67-L75). The same setup also registers WebSocket lifecycle and frame listeners in [`lines 79–85`](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/chromium/crNetworkManager.ts#L79-L85).

These are close to the events I accessed through Selenium's Chrome performance log in the previous article.

The difference is not that Playwright sees an entirely different network or anything like that. It consumes the browser events, handles their ordering and edge cases, and converts them into its own internal model.

## CDP does not deliver one tidy request object

It would be convenient if Chromium emitted one event containing a complete request, followed by another containing a complete response. It does not.

Request and response information can arrive through several protocol events. When interception is enabled, Playwright may need to correlate `Fetch.requestPaused` with `Network.requestWillBeSent`. Redirects alter request relationships. Service workers can cause expected events to be absent. Additional request and response headers arrive through separate events.

One awkward case appears in `_onRequestPaused`:

```ts
if (!event.networkId) {
  // Fetch without networkId means that request was not recognized by inspector, and
  // it will never receive Network.requestWillBeSent. Continue the request to not affect it.
  sessionInfo.session._sendMayFail('Fetch.continueRequest', { requestId: event.requestId });
  return;
}
```

See [`crNetworkManager.ts`, lines 245–249](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/chromium/crNetworkManager.ts#L245-L249).

Playwright cannot wait to correlate that paused request with `Network.requestWillBeSent`, because the source says the event will never arrive. It continues the request instead.

There is another special case for service workers. In `_onResponseReceived`, Playwright notes that frame-level requests handled by a service worker may never produce a `requestPaused` event. It may therefore construct the request when the response arrives. See [`crNetworkManager.ts`, lines 451–453](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/chromium/crNetworkManager.ts#L451-L453).

Header information has its own ordering problem. The source comment for `ResponseExtraInfoTracker` says the ordinary request and response events, and their corresponding `ExtraInfo` events, are dispatched through different unassociated channels and may arrive in any order. The tracker associates them so that the extra headers are reliably available by `requestfinished`. See [`crNetworkManager.ts`, lines 769–782](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/chromium/crNetworkManager.ts#L769-L782).

As you can probably gather, there are loads of edge cases like this. Handling them is not glamorous work, but it is a large part of why the public API feels simple compared with working directly against the protocol.

## Turning protocol events into Playwright events

Once Chromium's network events have been associated with a Playwright request, they pass into the page's `FrameManager`.

The request path marks the request as in flight, associates document requests with pending navigation, and emits the public request event:

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

That code is in [`frames.ts`, lines 329–343](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L329-L343).

Responses and completion events are emitted nearby:

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

The implementations, including request failure handling, are in [`frames.ts`, lines 345–356](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L345-L356).

This is how low-level CDP events become the public interface used by tests:

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

The in-flight request tracking lives in `frames.ts`.

When a request starts, Playwright adds it to the owning frame's `_inflightRequests` set. If it is the first active request, Playwright stops that frame's network-idle timer.

When a request finishes or fails, Playwright removes it. If the set becomes empty, Playwright starts the timer again:

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

The source is in [`frames.ts`, lines 385–403](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L385-L403).

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

See [`frames.ts`, lines 405–411](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L405-L411).

A favicon is browser housekeeping rather than meaningful application activity. An EventSource connection is long-lived by design, so counting it would prevent pages using server-sent events from ever reaching network idle.

The idle state is also calculated through the frame tree (the main page and its nested iframes). Lifecycle state is recorded on frames, and clearing lifecycle state resets the timer around the remaining current-navigation request. The relevant lifecycle and reset handling can be followed in [`frames.ts`, lines 573–584](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L573-L584).

This is honestly a more precise than saying Playwright waits until there are no requests. A closer description may be that Playwright tracks in-flight requests per frame, starts an idle timer when a frame's set becomes empty, and combines the resulting state through the frame hierarchy.

It also shows why `networkidle` is not a universal indication that an application is ready.

A quiet network tells us nothing about a CSS animation, a client-side timer, data already queued for rendering, or an application bug that prevented the expected request from starting.

## How the click starts waiting before navigation happens

There is another race to solve.

Suppose Playwright dispatched a click and only then began listening for navigation. A sufficiently fast navigation could start before the listener was installed.

The pointer action avoids that race by running the input inside `waitForSignalsCreatedBy`:

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

That method is in [`frames.ts`, lines 192–207](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L192-L207).

`SignalBarrier` acts as a gate. It keeps the action open until navigation initiated by the click has been acknowledged.

When a frame may request navigation, active 'barriers' are retained. When the possible request has been resolved, they are released. If a frame actually requests navigation, Playwright adds that frame navigation to the barrier. See [`frames.ts`, lines 169–189](https://github.com/microsoft/playwright/blob/ad18048db947ca0a47c7fa59b77718e3a06afafe/packages/playwright-core/src/server/frames.ts#L169-L189).

The barrier is not a general wait for every network request caused by the click. It coordinates browser signals created by the action, especially navigation. General network observation continues through the network manager and the public request events.

A click may trigger an API request without navigating. Playwright cannot safely infer which arbitrary response represents the business operation a test cares about. The test must express that relationship explicitly:

```ts
const responsePromise = page.waitForResponse(
  response =>
    response.url().endsWith('/orders') &&
    response.request().method() === 'POST'
);

await page.getByRole('button', { name: 'Submit' }).click();

const response = await responsePromise;
```

The response wait is created before the click for the same reason as the internal barrier: a fast event must not be missed.

This is why Playwright's documentation tells us to set up `waitForResponse` before the action that triggers it. The test applies the same broad principle Playwright uses internally with `waitForSignalsCreatedBy`: install the listener first, then perform the action.

## Connecting this back to Selenium

Following the source changes how I would describe my previous experiment, but it does not make the experiment wrong.

The Chrome performance log approach observed real CDP network events. Playwright's Chromium network manager listens to the same family of events.

What Playwright adds is a huge coordination layer. It installs protocol listeners, correlates Fetch- and Network-domain events, handles redirects and service workers, constructs request and response objects, exposes them as public events, tracks requests per frame, calculates network idle, and associates page-loading requests with navigation.

My Selenium examples addressed a narrower problem: determine whether network activity appeared to be in progress.

That may be enough for a particular test suite. Playwright needs a heavier implementation because a reusable framework must handle a broader range of browser behaviour, test styles, and edge cases than an application-specific helper.

A network counter also does not address the first half of the problem.

Before a request or navigation can exist, Playwright still has to make the click happen. That requires locator resolution, element-state checks, scrolling, geometry, hit testing, and browser input.

Network awareness cannot tell us that a transparent overlay is blocking a click. Pre-click actionability checks cannot tell us that the expected API response returned the correct data.

They solve different parts of synchronisation.

## Wrapping up

We began with one line:

```ts
await page.getByRole('button', { name: 'Submit' }).click();
```

Following it through the source reveals two connected systems.

The first prepares and dispatches the input:

```text
resolve the locator
check visible, enabled and stable
scroll the element
calculate a point
check the hit target
dispatch browser input
```

The second observes what the browser does next:

```text
receive raw request and response events from the browser
normalise them into Playwright's own objects
track which requests are still in progress
emit public events that tests can observe
associate page-loading requests with navigation
wait for navigation triggered by the action
calculate network-idle state across the page and its iframes
```

These are the browser and network signals I was trying to expose through Selenium in the previous article. Playwright uses the same underlying family of events, but it does considerably more than count requests.

CDP is noisy and sometimes incomplete. Related information arrives through separate events and in inconvenient orders. Requests may redirect, pass through service workers, come from cache, or fail before every expected event appears. Playwright contains the code required to turn that stream into a more stable model.

The implementation also shows why `networkidle` is only one possible signal. It represents a quiet period in which no tracked requests are active. It does not establish that the application is correct, useful, or ready for the next business-level assertion.

I still do not think the conclusion is simply to choose Playwright over Selenium.

Selenium users can observe CDP events, execute browser-side JavaScript, and build application-specific waits. Playwright has chosen to make more of that coordination part of the framework itself. How much you want buried under an interface is up to you. 

The useful lesson is to be precise about what we are waiting for.

Before a click, that may be a stable, enabled, and unobstructed target. After it, that may be a navigation lifecycle event or one specific API response. Sometimes network idle is useful. Sometimes it is the wrong question entirely.

I may still try reproducing Playwright's “is something blocking the click?” check in Selenium.

I will not be reproducing `crNetworkManager`. Life is too short!

Did I get something wrong? Have you followed a similar trail through Playwright or Selenium? Please get in touch at [joseph@josephward.tech](mailto:joseph@josephward.tech).
