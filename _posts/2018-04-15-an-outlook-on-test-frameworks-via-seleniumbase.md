---
layout: post
title: "An outlook on test frameworks (via SeleniumBase)"
date: 2018-04-15 19:51
tags: selenium python blog
---

It's frustrating to go from 0 to implementing automation in testing at any scale. For some, this is down to a lack of experience with programming or scripting languages, for others it's the pressure to get some testing work done and move on to the next priority. It could even be both. In any case, the effect of neglecting automation in software delivery is well-documented: increased risk.

One method of baffling these frustrations is to pick a framework that does some "heavy lifting" for you, usually in exchange for a prescriptive pattern for writing tests, less customisability or something like that. There's several "big" frameworks per-language, but I'm going to focus on SeleniumBase in this post because Python was my introduction to automation in testing many years ago and it's a useful jumping-off point.

## What is SeleniumBase?

[SeleniumBase](https://github.com/seleniumbase/SeleniumBase) is an open-source test framework maintained by Michael Mintz. Some of its features include:

* Built-in integrations with 3rd party tools (like Docker, AWS, etc.)  
* Simplified facade API over vanilla Selenium  
* Plugin support for other, commonly used testing modules (used for making reports, etc.)  
* Snazzy demo mode (probably suitable for wowing stakeholders)  

## Why are these features useful?

Take a look at this example that uses SeleniumBase:

~~~python
from seleniumbase import BaseCase


class SomeOtherTest(BaseCase):

    def test_some_other_thing(self):
        self.open("www.google.com")
        self.update_text("textarea", "text")
~~~

Now let's compare this to something similar using vanilla Selenium:

~~~python
import unittest

from selenium import webdriver


class SomeTest(unittest.TestCase):

    def test_some_thing(self):
        self.driver = webdriver.Chrome()
        self.driver.get("www.google.com")
        self.driver.find_element_by_css_selector("textarea").send_keys("text")
~~~

Your mileage may vary, but I think there's less cognitive overhead in understanding how to use SeleniumBase's single method of updating some text somewhere than vanilla Selenium, and frameworks like this typically wrap lots of common tasks into methods like this.

It also simplifies some web testing actions that, in my opinion, bewilder even (supposedly) experienced test developers.

Look at another example using SeleniumBase:

~~~python
from seleniumbase import BaseCase


class AnotherTest(BaseCase):

    def another_test(self):
        self.open("www.google.com")
        element = self.wait_for_element_visible("div.my_class", timeout=5)
~~~

Versus another, similar method using vanilla Selenium:

~~~python
import unittest

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support import expected_conditions
from selenium.webdriver.support.wait import WebDriverWait


class YetAnotherTest(unittest.TestCase):

    def yet_another_test(self):
        self.driver = webdriver.Chrome()
        self.driver.get("www.google.com")
        element = WebDriverWait(self.driver, 5).until(
            expected_conditions.visibility_of_element_located((By.CSS_SELECTOR, "div.my_class")))
~~~

Again, arguably simpler. 

# When are frameworks like this not useful?

Lots of test developers use frameworks like SeleniumBase to simplify their work and increase their productivitiy, but not all, and not for all projects. I definitely don't recommend relying too heavily on them. Their strengths, looked at differently, are also weaknesses. If you rely on built-in integrations then you're less likely to understand the nuances of these integrations yourself. Likewise, becoming too familiar with a framework's API may be a dead-end if it becomes unmaintained or you need to move onto something else (like a different project in another language).

Finally, if you write and maintain a framework yourself, you have a better understanding of what's happening under the hood. That means you may have some idea of what to do with any bugs you find, or how to extend and change how things work. 

I invite you to dig a little deeper into the how SeleniumBase's `wait_for_element_visible` method works to see what I mean by this, it's done differently to my example, you may prefer it or you may not.  

Put simply, these frameworks do indeed help with "heavy lifting", but sometimes you need to do the heavy lifting yourself.

# What to do next

You might have noticed that my examples are a bit of a "straw man". SeleniumBase's `wait_for_element_visible` example is easy to reuse, but so is my version if I extract a method iand abstract it away somewhere. DRY (Don't Repeat Yourself) is a principle in software engineering stated as "Every piece of knowledge must have a single, unambiguous, authoritative representation within a system", a key indicator of a robust framework. To see how this applies to a test framework I urge you to research the "page object pattern", which SeleniumBase is, of course, compatible with (and there's even an example buried somewhere in the documentation). 
