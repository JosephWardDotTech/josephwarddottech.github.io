---
layout: post
title: "Why Simple UI Tests Become Slow"
date: 2026-06-30 16:38
tags: blog
---


Sometimes, UI tests take longer than the code suggests they should.

A loop reads values from a table. A button takes two seconds to click. An interaction fails, is retried, and then passes.

When this happens, it is tempting to start with the code that looks 'busy'. Rewrite the loop, change the filtering, etc.

I wanted to know whether that was the right place to look.

To do so, I created a small page and used Playwright with Python to measure a few common patterns: reading data from collections, waiting for controls and interacting with elements that are replaced while the page is rendering.

## Reading a table

Let's imagine a table of orders. Each row contains an ID, status, age and value.

Our test needs to find the highest-value order that is awaiting approval and more than 30 days old.

One way to do that is to read every field into Python:

```python
rows = page.locator("[data-testid='order-row']")
orders = []

for index in range(rows.count()):
    row = rows.nth(index)

    orders.append(
        {
            "id": row.locator(
                "[data-col='id']"
            ).inner_text(),

            "status": row.locator(
                "[data-col='status']"
            ).inner_text(),

            "age": int(
                row.locator(
                    "[data-col='age']"
                ).inner_text()
            ),

            "value": money_to_float(
                row.locator(
                    "[data-col='value']"
                ).inner_text()
            ),
        }
    )

candidate = max(
    (
        order
        for order in orders
        if order["status"] == "Awaiting approval"
        and order["age"] >= 30
    ),
    key=lambda order: order["value"],
    default=None,
)
```

Filtering 100 dictionaries is obviously pretty cheap. Retrieving the values is where the time probably goes.

Four fields across 100 rows means about 400 calls for text. The loop looks local, although most of its work involves round trips with the browser.

Playwright has bulk methods that reduce those calls:

```python
ids = page.locator(
    "[data-col='id']"
).all_inner_texts()

statuses = page.locator(
    "[data-col='status']"
).all_inner_texts()

ages = page.locator(
    "[data-col='age']"
).all_inner_texts()

values = page.locator(
    "[data-col='value']"
).all_inner_texts()
```

The four lists can then be combined and processed in Python.

This is a simple improvement and keeps JavaScript out of the test. It does assume that the rows remain unchanged between calls, of course. On a page that updates frequently, the values in the four lists could stop referring to the same rows. Who knows. 

Another option is to extract each row in one operation:

```python
orders = page.locator(
    "[data-testid='order-row']"
).evaluate_all(
    """
    rows => rows.map(row => ({
        id: row.querySelector("[data-col='id']")
            .textContent.trim(),

        status: row.querySelector("[data-col='status']")
            .textContent.trim(),

        age: Number.parseInt(
            row.querySelector("[data-col='age']")
                .textContent,
            10
        ),

        value: Number(
            row.querySelector("[data-col='value']")
                .textContent
                .replace(/[^0-9.-]/g, "")
        )
    }))
    """
)
```

The browser returns a list of row data and Python performs the filtering.

Then again, if the test only needs one result, the filtering can happen in the browser too:

```python
candidate = page.locator(
    "[data-testid='order-row']"
).evaluate_all(
    """
    rows => {
        const candidates = rows
            .map(row => ({
                id: row.querySelector("[data-col='id']")
                    .textContent.trim(),

                status: row.querySelector("[data-col='status']")
                    .textContent.trim(),

                age: Number.parseInt(
                    row.querySelector("[data-col='age']")
                        .textContent,
                    10
                ),

                value: Number(
                    row.querySelector("[data-col='value']")
                        .textContent
                        .replace(/[^0-9.-]/g, "")
                )
            }))
            .filter(order =>
                order.status === "Awaiting approval"
            )
            .filter(order => order.age >= 30)
            .sort((left, right) =>
                right.value - left.value
            );

        return candidates[0] ?? null;
    }
    """
)
```

I measured all four approaches with 100 rows:

| Approach | Median |
|---|---:|
| Read each value separately | 1,437.49 ms |
| Four bulk collection calls | 22.12 ms |
| Extract all rows in one call | 11.83 ms |
| Return the selected row | 5.32 ms |

As you can see, the largest change came from stopping the individual reads.

Using Playwright's collection methods reduced the median from about 1.44 seconds to 22 milliseconds while leaving the filtering in Python.

One extraction call was quicker again. Returning a single result was the fastest version.

The figures do not mean 'every collection query should be written in JavaScript'. They show that the way data is retrieved can matter way more than the code used to filter it.

For a small collection, I'd just use ordinary locators. For a simple column, `all_inner_texts()` is probably enough. `evaluate_all()` becomes useful when several related values are needed from each row or when returning the full dataset serves no purpose.

## Waiting

Fixed sleeps are another common source of time in test suites:

```python
time.sleep(2)

page.get_by_role(
    "button",
    name="Approve"
).click()
```

A sleep added to fix an occasional failure is easy to understand. We've all done it. It also runs at full length whenever the test passes, which is unideal.

I tested a button that started visible but disabled, then became enabled after 200 milliseconds.

I compared three approaches:

```python
# fixed sleep
time.sleep(0.5)
button.click()
```

```python
# wait for visibility, then click
button.wait_for(state="visible")
button.click()
```

```python
# playwright magic autowait 
button.click()
```

The median results were:

| Approach | Median |
|---|---:|
| Sleep for 500 ms, then click | 542.21 ms |
| Wait for visibility, then click | 249.42 ms |
| Let `click()` wait | 249.82 ms |

All three completed successfully in every run.

The fixed sleep spent roughly another 292 milliseconds waiting after the button could already be used.

The visibility wait made almost no difference compared with calling `click()` directly. Strictly speaking, though, the button had been visible from the start. Its disabled state was preventing the action.

This is an easy mistake to make in test code. It's the sort of thing that's easy to miss because the wait itself succeeds. But a condition can be true and still be irrelevant to whatever the test is trying to do. 

An explicit wait for the correct condition would probably be more meaningful:

```python
expect(button).to_be_enabled()
button.click()
```

I would expect that to finish in roughly the same time as `button.click()` alone, with a small amount of assertion overhead. It makes sense when the enabled state is itself something the test wants to check. It depends on whether the 'enabled' state is part of whatever's under test or just a prerequisite for the action you're interested in. 

If that elements state isn't part of the test then this sort of thing just repeats a condition Playwright already considers before clicking.

All this to say, there is still a place for application-specific waits despite the "magic". 

Playwright can tell whether the browser considers an element visible, stable, enabled and able to receive events. It cannot know that an account balance has finished recalculating or that a background process has completed.

The useful distinction is between waiting for time to pass, waiting for a browser condition, and waiting for a testable condition. 

## When the page replaces an element

A page can replace a DOM node without appearing to change on the surface. 

I tested a button that began disabled. After 50 milliseconds, the page replaced it with a new button. The replacement became enabled 100 milliseconds later.

I compared a stored `ElementHandle` with a locator.

```python
button_handle = page.query_selector(
    "[data-testid='approve']"
)

button_handle.click()
```

```python
button = page.locator(
    "[data-testid='approve']"
)

button.click()
```

The results were:

| Approach | Median | Successful runs |
|---|---:|---:|
| Stored `ElementHandle` | 69.21 ms before failure | 0/6 |
| Locator | 193.28 ms | 6/6 |

The handle was quicker only because it failed as soon as Playwright discovered that the original node had been detached.

The locator succeeded because it could resolve the matching element again while waiting for the click to become possible.

In a real framework, the failed handle would usually lead to more work:

```python
try:
    button_handle.click()
except Error:
    page.locator(
        "[data-testid='approve']"
    ).click()
```

That recovery path adds another lookup, another click attempt and whatever framework code surrounds the retry.

Arguably, this benchmark is therefore not about which approach clicks faster. It shows that retaining a reference to one DOM node can create failure and recovery work when the page replaces that node (ie stale elements in other libraries). A locator avoids that path by describing how to find the current element.

This affects reliability first. It affects execution time once retries and recovery code are included.

## Choosing what to change

These examples became faster for different reasons. The table benefited from fewer browser calls. The button benefited from removing a fixed delay. The locator avoided a failure and retry path when the DOM node was replaced.

Browser side filtering will not help a test littered with sleeps. Better waiting will not help a loop that retrieves thousands of values one by one. A lazy locator will not make an expensive calculation disappear.

The original loop looked like the obvious thing to optimise, but turned out to be doing very little computation. Most of its time was spent asking the browser hundreds of small questions.

Did I miss something important? Am I crazy? Please let me know by getting in touch at [joseph@josephward.tech](mailto:joseph@josephward.tech).
