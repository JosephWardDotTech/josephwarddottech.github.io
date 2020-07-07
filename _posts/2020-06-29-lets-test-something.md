---
layout: post
title: "Let's test something"
date: 2020-06-29 08:26
tags: blog
---

How do I put this? Sometimes you’ve got to test. Not talk about testing, not agonise over planning, but get stuck into adding value as fast as possible. In this blog post I investigate the popular [restful-booker web service](https://github.com/mwinteringham/restful-booker). From first impressions to reviewing the shippable quality of the software. We'll imagine I've dropped into the project late, I know nothing about what's under test, and I've only got two hours. Let's see how it goes!

# Test ideation

First, I’ll create a test ideation mind map. My goal is to generate as many questions for my investigation as quickly as possible. I can’t collaborate with anyone to target project specifics, so instead I’ll lean on my own experiences with testing web services.

![Mind map of test ideation](https://josephward.tech/assets/img/image--001.png)

# Overall focus/themes

Using the above as a guide I will focus these 2 hours on:

1. **"Correctness"** (measured against any identifiable oracles)
2. **"Risk"** (measured against my professional experience)
3. **"Security"** (measured against exploitability of assets)

The intrinsic testability of this project will also help or hinder how productive this session will be (which is why tester involvement in these conversations early is so important).

As this project is using [mocha](https://mochajs.org/) as a test runner I have also identified two libraries that I will add to the dev dependencies of the project.

1. [istanbuljs/nyc](https://github.com/istanbuljs/nyc): a static analysis tool used to track unit test coverage. While I do not know of the efficacy of any existing tests, and there’s no reason to suspect a causal relationship between lack of tests and bugs, knowing what the coverage looks like will help focus my attention.
2. [rocha](https://github.com/bahmutov/rocha): a helper for mocha that randomises test order. Using this tool will help me identify where the tests may be interdependent (and so I where be less likely to trust test results).

Now then... time to get stuck in.

# Observations/questions

1. npm audit did not reveal any dependency issues
* **But** it configures dependencies to update to latest minor/patch version on `npm update` (and that’s not even considering dependencies of dependencies)
2. No licence specified in the package.json
3. Dockerfile line 26 runs `npm start`, which sets the `SEED` variable. `SEED` is looked up in line 11 of `routes/index.js` to add up to 10 pseudorandom bookings into the database when the applications starts?
4. No obvious (to me) way to take the app out of development mode – how will anyone deploy this easily? Dive into code to see what to do?
5. [nedb](https://github.com/louischatriot/nedb) is being used as an in-memory data store, which may mean as soon as the application stops running bookings will be lost?
6. counter in `models/booking.js` is just an incremented integer, meaning booking ids may be predictable/guessable?
7. The docs are generated from comments and not tied to code, which means they may slip easily (so probably not the best oracle?)
8. Tests themselves do appear to have pretty good coverage (according to istanbul nyc) so are probably a more reliable oracle?
* But tests aren’t abstracted in a massively reusable way (e.g. by domain object)
* But tests revealing some odd “intended behavior”, e.g. misused status codes a (201 back from a `DELETE`, arguably a 405 from `DELETE`ing something non-existent and not 404)
9. `responds with a subset of booking ids when searching for checkin and checkout date` seems a bit non-deterministic (passes sometimes, fails others) but I can’t figure out why (`beforeEach` hook should be blocking in the application/test code afaik due to the callback but maybe it’s not? need to check)
10. Payloads don’t seem to have any character limits on service, but what about db? 
11. XSS injection opportunities (as no filtering is happening)?
12. `/booking/:id` `PATCH` route not covered by any tests at all?
* Could be issues here!
13. Is it a great idea to embed an authorization key into the application?
14. Tests must have some inter-dependency somewhere, they aren’t consistently green when run out-of-order
15. (picky) but parts of this wouldn’t pass up-to-date linting rules from a style POV as there are missing semicolons, var is used, etc.
* Though sticking to a linter is good for readability/extendability anyway

# Conclusions
## Overall

I recommend restful-booker is **not** suitable for release at the current time because of how it would be deployed, improved upon, and kept secure. Observations **3**, **4**, **5**, **6**, **11**, and **13** need deeper investigation.

## My approach

I focused testing on **correctness**, **risk**, and **security**. I tested mostly by adjusting the `tests/spec.js` test file because of the time constraints.

## Assessing my approach

Although I probably missed quite a few functional issues (it’s a testing target so there are probably lots of gotchas) I feel I did a good job testing the application in actual terms. The unit tests looked adequate for the “happy bath”, therefore bugs were likely to be found in unexpected edge cases. Instead, I focused my investigation on “deeper” issues, like whether this application was suitable for release commercially and, if it was, what the consequences of that might be..

## My recommendations for future development/testing

1. Configuring a linter would have ensured code style was consistent
2. [Flow](https://flow.org/) could be introduced for type checking
3. Static analysis tools (like istanbul nyc) could have identified areas where test coverage could have been better
4. Testing tools like rocha could have been introduced to identify inter-dependency of tests
5. Bringing a tester in at the end of the process of releasing the application was of limited value (and expensive if any of the observations lead to rework)
6. No clear “line of thought” about how this was developed, what problem it was trying to solve, who made business logic decisions, etc.
7. Convenient access to some tool to make exploratory testing easier (e.g. a [postman](https://www.postman.com/) collection)
8. Testing seems focused on the highest level (actually taking CRUD actions), unit tests for logic may be a better place if the behavior will later be expanded (and some things should be possible to demonstrate without dependencies anyway)
9. The tests that do exist are not abstracted (e.g. into domain objects), which would have made them more readable

## Wrapping up

Although the exercise was time boxed, that did not include writing this blog post.

I enjoy exercises like this. Restriction breeds creativity and a timebox prompted me to find ways of multiplying my effort and focusing that effort on priority issues.

What do you think of the issues I raised? Did I miss anything obvious? What have you found in restful-booker? Please let me know via the available social links.
