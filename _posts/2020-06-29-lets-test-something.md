---
layout: post
title: "Let's test something"
date: 2020-06-29 08:26
tags: blog
---

How do I put this? Sometimes you’ve got to test. Not talk about testing, not agonise over planning, but get stuck into adding value as fast as possible. In this blog post I investigate the popular [restful-booker web service](https://github.com/mwinteringham/restful-booker). From first impressions to reviewing the shippable quality of the software. Let's assume I've dropped into a project at the eleventh hour, I know nothing about what's under test, and I've only got two hours. Let's see how it goes!

# Test ideation

I think mind maps are best used when talking in a group but I’m going to create one for test ideation out of habit and for readability. Alternatives could be physical notes, a action plan assembled from conversation with people already on the project, or simply letting this process happen more naturally from your own experience and familiarity with the type of thing under test.

![Mind map of test ideation](https://josephward.tech/assets/img/image--001.png)

# Overall focus/themes

Using the above as a guide I have decided to focus my effort across these 2 hours on:

1. **"Correctness"** (measured against identified oracles)
2. **"Risk"** (measured against my gut feel on how this might be iterated on)
3. **"Security"** (measured against exploitability of potential assets)

My heuristics will no doubt be informed by a deeper look into any oracles I discover. I will also need to lean heavily on my own experience of testing these sorts of projects.

I also plan to add the static analysis tool [istanbul nyc](https://github.com/istanbuljs/nyc) to the dev dependencies of my local copy of the project. The tool can assist me in quickly determining how much of the application code is covered by tests. It’s by no means a causal relationship, but the number of tests covering different parts of the application will help me identify potential problem stops (and not just “bugs” but potential opportunities for refactoring/improvement).

Now then... time to get stuck in.

# Observations/questions

1. npm audit did not reveal any dependency issues
* **But** dependencies are configured to update to latest minor/patch version on `npm update` (and that’s not even considering dependencies of dependencies)
2. No licence is specified in the package.json
3. Dockerfile line 26 runs npm start, which sets the `SEED` variable. `SEED` is looked up in line 11 of `routes/index.js` to add up to 10 pseudorandom bookings into the database when the applications starts
4. No obvious (to me) way to take the app out of development mode – how will anyone deploy this easily? Dive into code to see what to do? Problematic?!
5. [nedb](https://github.com/louischatriot/nedb) is being used as an in-memory data store, which means as soon as the application stops running bookings will be lost
6. counter in `models/booking.js` is just an incremented integer, meaning booking ids are predictable/guessable
7. The docs are generated from comments and not tied to code, which means they may slip easily (probably not the best oracle?)
8. Tests themselves do appear to have pretty good coverage (according to istanbul nyc) so are probably a more reliable oracle?
* But tests aren’t abstracted in a massively reusable way (e.g. by domain object)
* But tests revealing some odd “intended behavior”, e.g. misused status codes a (201 back from a `DELETE`, arguably a 405 from `DELETE`ing something non-existent and not 404)
9. `responds with a subset of booking ids when searching for checkin and checkout date` seems a bit non-deterministic (passes sometimes, fails others) but I can’t figure out why (beforeEach hook should be blocking in the application/test code afaik due to the callback but maybe it’s not?)
10. Payloads don’t seem to have any character limits?
11. XSS injection opportunities (as no filtering is happening)?
12. `/booking/:id` `PATCH` route not covered by any tests at all?
* Could be issues here!
13. Is it a great idea to embed an authorization key into the application?
14. Tests must have some inter-dependency somewhere, they aren’t consistently green when run out-of-order
15. (picky) but parts of this wouldn’t pass up-to-date linting rules from a style POV as there are missing semicolons, var is used, etc.
* Though sticking to a linter is good for readability/extendability anyway
16. Some tests may be inter-dependent? (discovered by using rocha)

# Conclusions
## Overall

If I were put in the position of investigating whether or not restful-booker was suitable for release as-is  I'd recommend **no**. While many of the issues I identified appeared to be minor there are deeper issues in how the application itself is configured/deployed. Observations **3**, **4**, **5**, **6**, **11**, and **13** in particular could lead to issues if this application were to be used commercially, and even if not give the potential of leaking sensitive information.

## My approach

I tried to focus testing around the three points identified in the “overall focus/themes” section, which were correctness, risk ,and security. For the most part, I tested by adjusting the `tests/spec.js` test file of the project itself due to the time constraints I imposed on myself.

## My methodology

Again, due to time constraints, I mostly used my own experience and the oracle of the existing tests to inform my methodology and guide heuristics. These included basic measurements such as “too big”, “different character sets”, “different inputs”, but at a higher level I used the static analysis tool ​istanbul to identify where there may not be sufficient test coverage of behavior to identify potential problem spots. I also used tools like [rocha](https://github.com/bahmutov/rocha) to tell me when the tests may be presenting an inaccurate representation of behavior.

## Assessing my methodology

Although I feel I probably missed quite a few functional issues in this application (how could I not it’s intended a technical testing bed?) I feel I did a good job testing this application in real terms. For the most part the tests looked good at covering the “intended” behavior of the application, therefore the bugs were probably to be found in behaviors that a user wouldn’t be expected to do in most circumstances. I instead focused my investigation on "deeper" issues, like whether or not this application was suitable to be released for a commercial purpose and, if it was, what the consequences of that might be in terms of iterating on it and the user’s security.

## My recommendations for future development/testing

1. Configuring a linter would have ensured code style was consistent.
2. As this is JavaScript something like flow could have been configured for type checking (which would become relevant as the application gets bigger)
3. Static analysis tools (like istanbul nyc) could have identified areas where test coverage could have been better
4. Testing tools like rocha could have been introduced to identify inter-dependency of tests
5. Bringing a tester in at the end of the process of releasing the application was of limited value (and expensive if any of the observations lead to rework)
6. No clear “line of thought” about how this was developed, what problem it was trying to solve, who made business logic decisions, etc.
7. Convenient access to some tool to make exploratory testing easier (e.g. a [postman](https://www.postman.com/) collection) may have made testing easier
8. Testing seems focused on the highest level (actually taking CRUD actions), unit tests for logic are arguably a better place if the behavior will later be expanded (and some things should be possible to demonstrate without dependencies anyway)
9. The tests that do exist are not abstracted (e.g. into domain objects), which would have made them more readable and more able to be extended

## My recommendations for future operation, maintenance, and extension

Some thought about implementing the suggestions in the previous answer would go a long way to helping with operability, maintainability, and extendibility. If nothing else I would particularly recommend suggestion 5, because the other suggestions may be considered to be symptoms of something like this as the root cause: conversations not happening earlier enough.

## Wrapping up

Although the testing part of this exercise was timeboxed to 2 hours I did go over that amount when finalizing this post as I needed some time to fix typos and refactor my language.

I did consider going a bit “meta” in this analysis by messaging [Mark Winteringham](https://twitter.com/2bittester) (the author of this service) on twitter and asking about the roadmap for restful-booker (for use as an oracle). Ultimately, however, I decided against it because then I could also conceivable introduce twitter posts and blogs into scope (as well as any issues in the project’s GitHub), which would be problematic.
