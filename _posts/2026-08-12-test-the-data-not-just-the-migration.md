---
layout: post
title: "Testing the Data, Not Just the Migration"
date: 2026-08-12
tags: blog
---

I recently finished working as Testing Tech Lead on a large, one off data migration, and I’m about to start thinking seriously about another. The biggest lesson I’m taking forward is that migration testing isn't just about proving the transformation code works but about understanding the data well enough to know what evidence would make that move acceptable.

Both involve data where getting it wrong has effects beyond an awkward support ticket. The first was data I’d call highly critical, and the next looks like it will be even less tolerant of mistakes.

The timing has made me reflect on what the first migration taught me and on what I would focus on sooner next time.

Interestingly, many of the tough testing questions weren't about the transformations themselves, but rather were about the data we were moving, how important it was, what assumptions had built up around it, and what evidence was needed before we could say those assumptions were still true after the move.

## Starting with the data

The obvious place to start with migration testing is the transformation itself. Given this input, do we produce the expected output? But it turned out to be only one part of the overall problem.

The harder questions were about data rather than code. What values exist in the source system? Which fields matter most? What assumptions, if any, have built up around them over time? What relationships need to survive the move? What counts as an ‘unacceptable change’?

Also, how complex a transformation is doesn’t say much about how assurance it needs. A very simple mapping still deals with data where losing, or slightly changing a small thing would cause real problems. So what matters is how important the data is, not the exact steps it goes through in what you're doing.

That sort of thinking pushed us to think about criticality first. Some fields needed to be preserved exactly, while others had rules about uniqueness, completeness or links between records. Some values were important because of what they meant elsewhere, not because the transformation itself was doing anything especially complex.

We also had to be clear about what an acceptable result actually looked like. For the data that matters most, it helps to know in advance which failures you can live with, which you can’t, and what should stop the migration instead of trying to fix forwards later.

Without this kind of thing it’s easy to end up with a bunch of impressive technical checks, but you lack the arguably more important shared understanding on whether the results actually mean anything.

## Understanding the data you actually have

A schema will tell you what a field could contain, but not what people have actually put in it over the last twenty years. In our case, we found values that were valid by the rules but carried meanings you wouldn't guess from the model, including sentinel values, odd defaults, padding, and field combinations that only made sense once someone understood the context/history behind them.

This is easy to miss if you start by writing tests against the spec, because you can end up proving the migration works for the data you expect rather than the data that really exists. The classic verification/validation trap. Data exploration and analysis therefore became part of the testing work, we checked value distributions, nulls, blanks, maximums, minimums, and values that appeared far more often than they should, then investigated outliers instead of simply saying they were ‘bad data’.

Something like `999` in a number field is a good example. Is this invalid, or it could be a twenty-year-old sentinel for “unknown” that some downstream processes still expect? Until you know which, there is no point writing any kind of test about it. You may even find entirely unexpected things that require new input from the business.

## Test data needs to resemble reality

Those kind of questions and output from data exploration fed directly into our test data. We began with static fixtures, where each test had its own small set of records containing exactly what it needed. This worked well at first, but became awkward as the suite grew. We ended up with lots of overlapping data, and it became really hard to test the migration at any useful scale.

Next, we centralised more of the data setup. This cut duplication and sped up the pipeline, but it also made individual scenarios harder to recreate because a failing test could depend on shared setup that wasn't obvious from the scenario itself. Eventually, we added generated data as well, which helped with volume and coverage by revealing field length issues, encoding problems and combinations we hadn't thought to write by hand.

The key thing was that the generated data was guided by what we had learned from the real thing. Faker can generate believable names, dates and addresses, but it cannot infer years of undocumented business rules, so believable data isn't always representative data. For another migration, I would start earlier with a mix of exploration, generated data and targeted fixtures: generated data gives you volume, while handcrafted cases still matter when you need to understand a behaviour exactly.

## Making the migration easier to test

We found it helped to make the migration itself easier to inspect. As data moved through a component, the component already knew a lot about what was going on. How many records it received, how many it changed, which ones it rejected, and where the output went. It therefore made sense to expose that information directly, rather than forcing tests to rebuild it later.

Components counted what they received, processed and wrote, while unexpected values were logged and records that couldn't be transformed were easy to find, with the reason shown. None of that proves the code is correct, but it does make the behaviour much easier to test. If 10,000 records go into a stage and 9,997 come out, I want to know what happened to the other three without having to reverse engineer the run afterwards.

This kind of self-checking also gives you another set of data to compare with your external checks!

## Checking everything you reasonably can

Some checks can be done across the whole of the data fairly inexpensively, and record counts, fixed mappings, duplicate checks, referential integrity and known invariants are examples of that. When those checks are available, it usually makes more sense to run them across everything rather than rely on a sample. Sampling is useful, but mainly when checks are expensive or impractical to run on every record, or when you want an independent check on automated controls (i.e. manual belt and braces).

One thing I learned about sampling is that a sample size on its own doesn't mean that much. “We checked 3,000 records” sounds reassuring, but it does not tell you what that sample actually means. For example, if you inspect 3,000 random records and find no defects, a 95% Clopper–Pearson interval gives an upper bound of about 0.1% for the true defect rate. That is useful, but it is also worth viewing it in the context of the whole dataset: on millions of records, 0.1% equals tend of thousands of records, and whether that is acceptable depends completely on what those records actually mean. You'll learn to love statistics!

This is also where acceptance sampling is generally better than picking a random number. Start with a defect level you can tolerate and the risk you can accept, then design the sample to match. The point is to set the threshold before you see the results, because if you only decide what is acceptable after seeing the sample, you're obviously are not really using the sample as a real acceptance test.

## Being clear about what your tests don't do

We made similar tradeoffs around external services. Early on, we stubbed some dependencies, which made sense because we didn't want our pipeline tied to other teams’ environments or held up by unrelated hiccups. That approach gave us fast, predictable tests of our own behaviour.

Later, though, we needed to learn things the stubs couldn't tell us, including batching, retries, connection behaviour and what happened under load. The takeaway wasn't that this approach was wrong, it was that the stubbed path only proved a limited set of things. Every test boundary removes some complexity, but it also removes some evidence you may want. Next time I’d still stub external dependencies early, but I’d add a real integration path sooner, even if it ran less often.

## Some failures only exist in real life

Some parts of the migration relied on database state, network setup, credentials and infrastructure that were hard to reproduce fully on the dev side. That meant some failures only showed up once the pipeline ran. The same sort of thing applies to migration performance, where smaller local datasets and simpler infrastructure could tell us if the logic worked, but they didn’t show how the migration behaved with larger volumes, real network paths, distributed components, connection limits and productionlike activity. Part of testing therefore became practical performance testing to find where the migration needed tuning before we trusted it at scale.

We couldn't eliminate environment specific failures entirely, so we added tests and logging to make diagnosis easier. Tests that run individually in CI and failing tests include diagnostic details so we dont need another commit to understand failures. The same feedback aided performance tuning, so when a run slowed, timed out, or behaved differently under load, the details better showed where the holdup actually was.

If some failures or bottlenecks only show up in a particular environment then being able to see them clearly should be part of the test design. For us, that meant treating production like runs as both functional and performance checks. They show whether the migration still gave the right result, and whether it could do so reliably, in a realistic time, on infra that acted more like the place it would actually run.

## Temporary code changes some tradeoffs

The temporary nature of the migration did change some of our coding decisions. We accepted more duplication than I normally would, kept some configuration near where it was used, and skipped creating abstractions that weren’t worth the extra clicking around. That was all deliberate: this wasn’t a framework anyone was expected to maintain for the next five years. When something failed during a rehearsal or a production like run, being able to read a scenario and see exactly what it set up was often more useful than removing every repeated line. DAMP beats DRY here, but there’s still a line between deliberate duplication and a mess; temporary software can reasonably shift that line. What it shouldn’t change is the assurance outcomes.

## Beyond one off migrations
Even though this was a one off migration, most of the testing ideas apply more generally.

Regular migrations, sync processes, ETL jobs, backfills and normal app integrations all face the same kind of issues. Data moves across a boundary, people make assumptions about it, and someone has to check if those assumptions are still true afterwards.

The transformation code is part of that issue, but rarely the whole thing.

Have you ever worked on a data migration? Do you agree or disagree with my takeaways? Please get in touch at joseph@josephward.tech either way, I love talking about this stuff.
