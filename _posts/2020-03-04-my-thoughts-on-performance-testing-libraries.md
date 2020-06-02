---
layout: post
title: "My thoughts on performance testing libraries"
date: 2020-06-01 20:45
tags: performance blog
---

Sometimes I write things in Slack conversationally that might be useful to refer to later or reflect on. Going forwards I'll try to remember to add them here.

![Image of my thoughts on performance testing libraries](https://josephward.tech/assets/img/Screenshot%202020-06-01%20at%2020.43.16.png)

There are some open source proxies for tools that don't appear to have one at first glance (like locust.io and artillery).

locust: https://github.com/zlorb/locust.replay
artillery: https://github.com/artilleryio/recorder

You could also use something like https://github.com/Carlgege/HAR-JMX-JmeterConverter to convert a HAR file into JMX if you don't want to set up the proxy for whatever reason. Plenty of options exist if you're willing to delve into open source software (and potentially work with something a bit out-of-date).

Overall the best advice I can give for anyone stepping into performance testing is to think about what you're testing, how, and why. Just like with UI testing I find it's best to be as precise as possible. Don't worry about testing things you don't need to (SSO blocking you? Find out if you can turn it off). In many ways performance testing is a great exercise in figuring out what's important to see in a test and what's not. If you don't believe me start by using a proxy and you'll see just how much stuff is actually involved with loading even relatively simple-looking websites! 

Example: consider a static UI that's getting all of its content through a REST API. There's not much point including the requests for html, js, images, etc. That's probably not where you'll find things. The API routes serving that content to the frontend is where the real performance information is.

Even with that said the raw response times any tool shows you is only part of the picture. You're going to get an equal if not greater amount of information from seeing how the host of the API reacted while the simulation was taking place... memory consumed, if a spike coincided with more waiting for the client, how quickly things settled down etc. 
