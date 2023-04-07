---
layout: post
title: "Green Testing: Measuring and Reducing Your Software's Energy Use and Emissions"
date: 2023-04-07 21:10
tags: blog testing sustainability
---

In addition to verifying software functionality, testing is where we assure that we meet our non-functional needs. If we treat emissions as a non-functional need, is it an appropriate place to hold ourselves accountable for our software's energy use and emissions?

What constitutes a test pass or fail? There often won’t be a quantifiable number that you can test an emissions pass/fail against, at least not until green software development becomes a much more mature field. But this doesn’t mean we can’t still measure, baseline, and understand the impact of new features and releases on the overall emissions picture.

# DETECTING UNEXPECTED CHANGES
Given that we most likely don’t understand what constitutes a pass/fail for emissions, can we baseline an application’s energy use and emissions in test, and then use this baseline to understand the relative change against the baseline of future releases?

*To create a good baseline, we would need to measure or model a few things at least, for example:*

* Cloud energy use and emissions (including embodied) using cloud provider tools, or the cloud carbon footprint tool
* Non-cloud hardware energy use and emissions (including embodied carbon) using watt-hour meters or energy profilers
* End-user device energy use (and embodied carbon) using a watt-hour meter, an energy profiler, or a SaaS tool such as Greenspector (which runs web pages on real mobile devices and measures energy use). In many cases, this is vital as end-user devices may significantly affect software energy use. Hardware power monitors are typically the most accurate way to measure end-user device energy use; however, they require the disassembly of mobile devices to connect directly to the power source.
* The Green Software Foundation has already developed an early version of a specification for measuring software’s carbon intensity - the SCI Specification, as well as some [guidance](https://greensoftware.foundation/) (note, this is still in development).

While the equation is relatively simple, it’s not necessarily easy to calculate the numbers needed, and there is room for plenty of interpretation. The score is designed for benchmarking. As long as it is calculated consistently between releases, it will be sensitive to changes in the carbon intensity of your software caused by green software actions – demonstrating the impact that the changes between each calculation have had.

With a measurement approach and baseline in place, it should be possible to alert to unexpected changes proportional to the baseline as part of the test approach. But this might not be easy to fully automate today, given the currently available tools and need for hardware meters.

Another worthwhile benefit to using the SCI Specification for your internal measurement is the ability to publish your calculation, evidencing both the green credentials of your software today and the impact of improvements you have made in your releases. This could be a powerful way to show potential users that you are making sustainability an essential part of your work.

# CHANGE AWARE TESTING
In modern Agile development, tests are automated. There are significant test packs at each testing level, and whenever code is changed, the tests are often all run to verify that change. This goes from running unit test packs on a developer machine to running integration and UI tests (among others) during a CI build.

Each of these runs has an energy cost. For large enterprises with many large software solutions, datasets, and test suites, the impact could be significant (in energy, emissions, and time) – it’s certainly not rare for full regression packs to take hours to run.

The ultimate example of this is the testing approach taken by Meta and Google. For example, Google has developed a change-aware testing system for its Test Automation Platform, which carries out more than 150 million daily test runs. Even with Google’s massive compute resources, it was impossible to test code commits individually, and its solution limits test execution based on code changes. As well as widening a bottleneck, this will significantly impact energy use and emissions.

Approaches to change aware testing run from tagging tests to control, which are run depending on the context of code changes (e.g., run all tests that exercise the shopping cart functionality), through to Test Impact Analysis and even Predictive Test Selection at the advanced end of the spectrum.

Pioneered by Microsoft, Test Impact Analysis analyzes the call graph when running tests and uses this analysis to determine which tests to run to exercise future production tests.

Predictive Test Selection uses machine learning to predict which tests to run based on an extensive historical dataset of test outcomes – for example, see [this research paper by Meta](https://research.facebook.com/publications/predictive-test-selection/).

Test Impact Analysis is available in Visual Studio, and you can apply it to Azure Pipelines. Also, there are testing and CD frameworks that support this approach. You could even implement something yourself. Predictive Test Selection is just starting to reach availability in tooling such as Gradle Enterprise.
