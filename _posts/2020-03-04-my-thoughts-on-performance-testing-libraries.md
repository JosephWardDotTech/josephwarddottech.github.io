---
layout: post
title: "My thoughts on performance testing libraries"
date: 2020-06-01 20:45
tags: performance blog
---

Sometimes I write things in Slack conversationally that might be useful to refer to later or reflect on. Going forwards I'll try to remember to add them here.

![Image of my thoughts on performance testing libraries](https://josephward.tech/assets/img/Screenshot%202020-06-01%20at%2020.43.16.png)

Edit: 

For anyone reading this who is interested, there are some open source proxies for tools that don't appear to have one at first glance (like locust.io and k6).

locust: https://github.com/zlorb/locust.replay
artillery: https://github.com/artilleryio/recorder

You could also use something like https://github.com/Carlgege/HAR-JMX-JmeterConverter to convert a HAR file into JMX if you don't want to set up the proxy for whatever reason.

The best advice I can give for anyone stepping into performance testing is to think about what you're testing and why. Just like with UI testing I prefer to be precise. Don't worry about testing things you don't need to.

Example: consider a static UI that's getting all of its content through a REST API. There's not much point including the requests for html, js, images, etc. That's probably not where you'll find things. The API routes serving that content to the frontend is where the real performance information is.

Even with that said the raw response times any tool shows you is only part of the picture. You're going to get an equal if not greater amount of information from seeing how the host of the API reacted while the simulation was taking place... memory consumed, if a spike coincided with more waiting for the client, how quickly things settled down etc. 
