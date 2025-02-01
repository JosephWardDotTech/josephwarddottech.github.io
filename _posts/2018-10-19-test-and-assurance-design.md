---
layout: post
title: "Lessons Learned: Revisiting Test and Assurance Design for Past Projects"
date: 2018-10-19 21:38
tags: testing practice blog
---

Recently, I commiserated with a developer on missed opportunities in past projects. This was a great chance to spitball ideas for test and assurance design. What follows is a bit biased towards righting old wrongs, but may be good enough to revisit for future projects. Anyway, this is what I came up with:

> "Ideally, we'd have discussed the context of the change and implied/expressed requirements ASAP with whoever is promoting that changek. From there, I'd try to have input on and review any unit/integration tests and we’d discuss the different ways to spin criteria and where issues might creep (due to gaps in understanding). That, in turn, would inform any test coverage at different levels and what needs to be exploratory tested. <br><br> If that testing found anything, we'd have another conversation about where the best level to apply fixes/check for regressions are (if that's relevant). If there was are any browser tests, we’d talk about the specific aims and how often they run. Also, what the indicators risk are, e.g. changes to other components, changes to client requirements (e.g. browser upgrades), etc. <br><br> I’ve always preferred to be a light touch and not actually write that much code at all if I can avoid it. Because I’m not _really_ employed to write code, I’m employed to test things and promote quality*...<br><br>\* whatever that  is, but that's a separate discussion"

This approach is pretty much geared towards conservating effort. It's a deliberate choice to try and catch things earlier, not later. The tester is having conversations, solving problems, and actually testing as much as possible. I can see this annoying some of my colleagues who quite like spending their days writing their own automation code, and I'm not totally throwing that idea out, but pretty much only when it's appropriate (which it might not always be every).

Taking this approach at face value, what's jumping out at me straight away (but I didn't know of at the time) is "efficacy". I read [a blog post from Google recently](https://testing.googleblog.com/2018/09/efficacy-presubmit.html) that brings that term into a testing context as follows:

> Originally named "Test Efficacy", a small team was formed in 2014 to quantify the value of individual tests to the development process. Some tests were particularly valuable because they provided a reliable breakage signal for critical code. Some tests were not useful because they were non-deterministic or they never failed. Confoundingly, tests would change in value over time as well. The team's initial intention was to present this information to developers and help them optimize the development process.

I definitely think analysis of the efficacy of a test (whenever that happens -- though I suppose continually!) is something that could, and should, live in the assurance ecosystem. It's definitely something I want to investigate and a talk about with my colleagues going forwards.

So... if you're reading this, what do you think? Anything you'd change? Any parts you really like? Please feel free to get in touch via the usual channels if that's the case, I'd love to speak with you. 
