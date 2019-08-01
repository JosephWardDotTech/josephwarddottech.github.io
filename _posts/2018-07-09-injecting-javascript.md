---
layout: post
title: "injecting javascript for testability"
date: 2018-07-09 09:51
featured_image: 
code: true
tags: javascript testability blog
---

Sometimes, you can test things on a webpage by injecting JavaScript. It's fairly simple, fairly powerful, but not without its gotchas. So what can you do and how? Here's a trivial example:

# Example 1 - Basic injection

~~~html
<button id="submit" type="submit" onclick="sendRequest()">Submit</button>
~~~

~~~js
function sendRequest() {
  // Something that sends a request.
}
~~~

So you click the button, it sends a request. Simple right? There are many ways you can test this, but I rarely see people inject JavaScript (which you can either do either programmatically or via your browser's console).

~~~js
var someNumber = 0;

document.getElementById("submit").addEventListener("click", function() {
  someNumber++;
});
~~~

Now click the button.

~~~js
$ someNumber == 0; // this will be false
~~~

So what happened here? We instantiated a variable as 0. We used the document object `addEventListener` method to increment that variable `onclick`. We checked the variable's value was 0 and it wasn't. That shows, at the very least, `sendRequest` was called. 

# Example 2 - Less basic injection

This also works.

~~~js
var someNumber = 0;

var _sendRequest = sendRequest;
sendRequest = function() {
  _sendRequest;
  someNumber++;
};
~~~

When you click the button.

~~~js
$ someNumber == 0; // this will be false
~~~

We create a var that captures the original `sendRequest` (1). We override `sendRequest` (2) with that variable and increment `someNumber` (3). Now, when we invoke `sendRequest` (2) it does both (1) and (3)!

# Example 3, 4, and 5 - Something you might be able to use

Okay, interesting but not very useful so far. But let's say I'm one of the many websites that use the Navigator.geolocation API. If my visitor is in the EU, I block them from seeing the site. Annoying? Yes, but, as it turns out, avoidable.

~~~js
var customPosition = {};
customPosition.coords = {};
customPosition.coords.latitude = 40.71;
customPosition.coords.longitude = 74.0;
navigator.geolocation.getCurrentPosition = function(success, error) {
  success(customPosition);
};
~~~

Now, as far as the website knows, I'm visiting from New York. Neat!

If you inject the JavaScript early you can also inject something like my second example to filter the `beforeunload` listener (which has tripped up my UI testing frameworks I don't know how many times).

~~~js
var _addEventListener = window.addEventListener;
window.addEventListener = function(type, listener, options) {
  if (type !== "beforeunload") {
    _addEventListener(type, listener, options);
  }
};
~~~

Got an all-singing, all-dancing webapp where animations sometimes don't resolve before you try a `click` an element? No reliable way of figuring out when the animations end? Why not try something like this.

~~~js
(() => {
  var node = document.createElement("style");
  node.innerHTML =
    "*, *:before, *:after {transition-property: none !important;" +
    "transform: none !important;animation: none !important;}";
  document.body.appendChild(node);
})();
~~~

We create a `style` tag (used to, as the name implies, style HTML) with references to CSS animation properties, set those to `none`, then mark that as `!important` meaning it'll be listened to over of other instructions.

Cool, huh?

# wWat to do next

So if this works then why aren't more people doing it? Well, here's the rub. A lot of the time `window`, where we'll be injecting this JavaScript from, won't have what we want to override in scope. Worse still, we often won't be able to bring what we want into scope as it'll be within a closure. If that's the case, overriding is more like cracking a walnut with a sledgehammer than the surgical modification we'd like to make. Changing the code that a user's computer will interpret for a test can also be hard to justify. Finally, code minification and obfuscation libraries frustrate this by design.

So does injecting JavaScript have a place in the modern testers toolkit? Despite the complications, I'd definitely agree. Yes, there are other ways of doing the same thing; test doubles being the obvious one if your focus is automation. They key is, as always, context. If you're modifying exercised code to get to unexercised code that feels less risky to me than not testing at all, particularly if it's not covered at a lower level (somehow). For exploratory testing or ad-hoc test doubles, though, it's definitely worth investigating.

**Edit:** this post was edited 13/07/18 to include an animation cancelling injection that I had previously forgotten about.
