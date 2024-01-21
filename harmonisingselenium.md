---
layout: page
title: Harmonizing Selenium with Playwright and Cypress: A Journey Through Network Event Handling
description: >
  Dive into the world of network event control in software testing. Discover how Playwright and Cypress provide granular control over network events and learn how to mimic their approaches using Selenium. Enhance the reliability and robustness of your test automation frameworks with insights and code examples.
hide_description: true
menu: false
---

# Introduction
Playwright and Cypress are well-known for their 'autowait' feature. This neat trick aligns elements seamlessly and avoids many 'test flake' issues, like Selenium’s 'StaleElementException' and 'NoSuchElementException.' However, I'm skeptical about this so-called 'magic' in testing. A solid test should be free of surprises, including variables that are supposed to smooth out the process. I don't accept hidden logic without scrutiny - I believe in dissecting and analyzing everything to ensure our tests are reliable. A deep understanding is also invaluable for troubleshooting and grasping what the tests are actually doing with whatever's under test.

This post will reveal a particular feature of these tools that I think is quite helpful. I'll cover how this feature can be included in current Selenium frameworks to enhance the trustworthiness of your tests. It's all about having specific control over **Network Events**.

# Test Flake
"Test flake" means when testing gives you different results each time. People often think Selenium, a testing tool, has this problem, but it's not really unreliable by nature. Selenium follows the W3C WebDriver rules and uses web commands to control the browser. This method can sometimes cause small delays and chances for errors between starting an action and finishing it. Also, Selenium has ways to wait for things to happen, but it doesn't automatically sync like newer tools do. This means testers have to do more work themselves. Plus, when you mix different ways of testing and complicated web designs, it seems like Selenium is more unreliable than it actually is. But really, Selenium is doing its job as it's supposed to, within its own limits.

Playwright connects directly with browser tools for developers and debuggers. This lets it control browser actions very precisely and quickly. Cypress operates differently; it works alongside the application's run-loop. It acts as an intermediary for network communications and runs concurrently with the browser's main thread

# Playwright
As mentioned, Playwright's essence lies in its seamless integration with the browser's developer and debugging protocols. It listens to browser lifecycle events, checks for their completion, and provides an array of tools for controlling and automating browser actions. 

![An architectural diagram of Playwright](https://josephward.tech/assets/img/playwright-arch.jpg)

Image credit: [ProgramsBuzz](https://www.programsbuzz.com/).

Putting aside the left hand side of the diagram for now, notice how the Playwright NodeJS server is shown to use CDP. CDP Stands for Chrome DevTools Protocol. What's that?

From the CDP [documentation](https://chromedevtools.github.io/devtools-protocol/):
> The Chrome DevTools Protocol allows for tools to instrument, inspect, debug and profile Chromium, Chrome and other Blink-based browsers.

> Instrumentation is divided into a number of domains (DOM, Debugger, Network etc.). Each domain defines a number of commands it supports and events it generates. Both commands and events are serialized JSON objects of a fixed structure.

Once Playwright has established and integrated the event listeners effectively within its framework — a process that involves a fair amount of complexity so won't be covered in full — it gains the capability to perform a variety of actions within the browser. 

Ultimately, once made user friendly in the higher-level, user accessibke Playwright API, this allows Playwright to expose this functionality to users as seen in the [Network events](https://playwright.dev/docs/network#network-events) part of the documentation. A similar pattern can be seen in how Playwright interacts with many other things in the CDP spec.

Note: the CDP is why, at launch, Playwright only supported Chromium based browsers. As time has gone on, non-Chromium based browsers are supported by analagous systems for communicating with the broser at a low-level in a similar way. But at the time of writing, there are still some features missing from these implementations (such as modifying requests on the fly, performance data, heap snapshots, etc).

# Cypress
By contrast, Cypress achieves this level of control by proxying network events through itself.

![An architectural diagram of Cypress](https://josephward.tech/assets/img/cypress-arch.jpg)

Image credit: [https://www.tutorialspoint.com/index.htm](Tutorialspoint)

Because Cypress stands in between the browser and the Internet it therefore affords Cypress complete control over associated events, notifying itself when events are dispatched, received back, allowing for manipulation of requests, connection speed profiling, etc. 

# Selenium

Selenium, by contrast, has only fairly recently supported access to the CDP in Selenium 4. In my personal opinion, Selenium's tone is generally also quite dismissive of the CDP. I suppose this is generally because Selenium's approach has and continues to be that waiting for things specifically visible to the user to determine when to take actions is the best course of action. Personally, however, I think this harkens back to a simpler time of the internet, before frameworks like React made (for better or worse) pages more dynamic, or utilised a shadow DOM elusive to Selenium, or what have you. More information on that can be found [https://www.tutorialspoint.com/index.htm](here). 

# Mimicking Playwright's Approach With Selenium

*Note: all of my examples will use Python for readability, but it's obviously possible to do with other Selenium bindings as well.*

As already discussed, the CDP allows for very granular control over network activity. Here, we are using it to monitor when requests are sent and when requests are sent. Selenium has various ways of using CDP, logging events just being one, which also allow you to do interesting things like modify outgoing requests and incoming responses, simulate network conditions, etc.

```python 
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.desired_capabilities import DesiredCapabilities
import time
import json

# your path to chromedriver
chrome_driver_path = './chromedriver' 

# enable Chrome performance logging for granular network event information (using CDP)
capabilities = DesiredCapabilities.CHROME
capabilities['goog:loggingPrefs'] = {'performance': 'ALL'}

# setup chrome
service = Service(executable_path=chrome_driver_path)
driver = webdriver.Chrome(service=service, desired_capabilities=capabilities)

# navigate to desired website
driver.get("https://josephward.tech/")

# initialise an empty set for keeping track of open/closed network events
sent_requests = set()
received_responses = set()

# initialise a timer 
start_time = time.time()
timeout = 30

while True:
    # check in log for events
    logs = driver.get_log('performance')
    for log in logs:
        event = json.loads(log['message'])['message']
        method = event['method']

        if 'params' in event and 'requestId' in event['params']:
            request_id = event['params']['requestId']

            # if method of event is Network.requestWillBeSent add its ID to sent requests
            if method == 'Network.requestWillBeSent':
                sent_requests.add(request_id)

            # if method of event is Network.responseReceived add its ID to sent requests
            elif method == 'Network.responseReceived':
                received_responses.add(request_id)
                if request_id in sent_requests:
                    # log and print matched events and outstanding from total
                    print(f"Matched Request and Response: {request_id}")
                    print(f"Remaining unmatched requests: {len(sent_requests) - len(received_responses)}")

    # check if all requests have received responses by comparing set. if matched, end program
    if sent_requests == received_responses:
        print("All requests have been matched with responses.")
        break

    if time.time() - start_time > timeout:
        print("Timed out waiting for responses.")
        break

    time.sleep(0.5)  # rerun check every interval

# finally, close the browser
driver.quit()
```

# Mimicking Cypress' Approach With Selenium

Cypress creates its own proxy to send network requests through. By installing BrowserMob proxy we can mimic this. BrowserMob proxy also has an API for granular network interception that allows us to rewrite requests, responses, and other things just like CDP. 

```python
from browsermobproxy import Server
from selenium.webdriver.chrome.service import Service
from selenium import webdriver
import time
import hashlib

# your path to browsermob proxy and chromedriver
bmp_path = './browsermob-proxy-2.1.4/bin/browsermob-proxy'
chrome_driver_path = './chromedriver' 

# start the proxy
server = Server(bmp_path)
server.start()
proxy = server.create_proxy()

# setup chrome to use proxy
options = webdriver.ChromeOptions()
options.add_argument('--proxy-server={0}'.format(proxy.proxy))
options.add_argument('--ignore-certificate-errors')
service = Service(executable_path=chrome_driver_path)
driver = webdriver.Chrome(service=service, options=options)

# create a HTTP Archive Format (HAR) 
proxy.new_har("test")

# navigate to desired website
driver.get("https://josephward.tech/")

# initialise an empty set for keeping track of open/closed network events
sent_requests = set()
received_responses = set()

# initialise a timer 
start_time = time.time()
timeout = 30

while True:
    # get HAR log
    har_log = proxy.har['log']

    # process entries in HAR log
    for entry in har_log['entries']:
        request = entry['request']
        response = entry['response']
        
        # add unique identifier to each entry
        request_details = f"{request['url']}_{request['method']}_{entry['startedDateTime']}"
        request_id = hashlib.md5(request_details.encode()).hexdigest()

        # add request to set (checking if it's there) then log it
        if request_id not in sent_requests:
            sent_requests.add(request_id)
            print(f"Request Sent: {request_details}")

        # add response to set (checking if it's there) then log it
        if response['status'] and request_id not in received_responses:
            received_responses.add(request_id)
            print(f"Response Received: {request['url']} with status {response['status']}")

    # print progress
    print(f"Total Requests Sent: {len(sent_requests)}")
    print(f"Total Responses Received: {len(received_responses)}")
    print(f"Remaining unmatched requests: {len(sent_requests) - len(received_responses)}")

    # check if all requests have received responses
    if sent_requests == received_responses:
        print("All requests have been matched with responses.")
        break

    if time.time() - start_time > timeout:
        print("Timed out waiting for responses.")
        break

    time.sleep(0.5)  # rerun check every interval

# finally, close the browser and stop the proxy
driver.quit()
server.stop()
```

# A Third Way - Monkey Patching 
By injecting JavaScript we can monkey patch the browser's internal methods for sending requests and receiving responses. This allows us to extend them on the fly with whatever extra code we want. While this is quite powerful, you will notice that the JavaScript should be injected after the page has loaded. Full page navigation will also reset the injected JavaScript, meaning this is typically only useful on single page applications, on features within a web application that won't cause page navigation, etc. So it has both flexibility but fairly obvious limitations.

```python
from selenium.webdriver.chrome.service import Service
from selenium import webdriver
import time
import hashlib

# your path to browsermob chromedriver
chrome_driver_path = './chromedriver' 

# Setting up Selenium 
options = webdriver.ChromeOptions()
options.add_argument('--ignore-certificate-errors')

# setup chrome
service = Service(executable_path=chrome_driver_path)
driver = webdriver.Chrome(service=service, options=options)

# navigate to desired website
driver.get("https://josephward.tech/")

# define and execute the XMLHttpRequest monkey patch
js_script = """
window.openedRequests = {}; // initialise empty object for tracking
window.closedRequests = {}; // initialise empty object for tracking

(function() {
  var oldSend = XMLHttpRequest.prototype.send; // we retain a reference to the original 
  var oldOpen = XMLHttpRequest.prototype.open; // we retain a reference to the original 

  // Intercept XMLHttpRequest open method to assign a unique ID and track the request
  XMLHttpRequest.prototype.open = function(method, url) {
    this.requestId = Math.random().toString(36).substr(2, 9); // Generate a unique ID
    window.openedRequests[this.requestId] = {
      url: url,
      method: method,
      content: null
    };
    return oldOpen.apply(this, arguments); // using original reference
  };

  // Intercept XMLHttpRequest send method to track when the request is complete
  XMLHttpRequest.prototype.send = function() {
    var requestId = this.requestId;
    this.onreadystatechange = function() {
      if (this.readyState === 4) { // this readyState means the request has completed
        window.closedRequests[requestId] = {
          url: window.openedRequests[requestId].url,
          method: window.openedRequests[requestId].method,
          content: this.responseText
        };
      }
    };
    return oldSend.apply(this, arguments);
  };
})();
"""

# execute the JavaScript code on the page
driver.execute_script(js_script)

# wait for some time for the page to load (note: you shouldn't do this in a real test, i am doing it here for demo purposes!)
time.sleep(5)

# click a link
element = driver.find_element("xpath", "//*[contains(text(),'Tech Talks')]")
driver.execute_script("arguments[0].click();", element) # my website is silly so i need to do a jsclick. sorry

timeout = 30 
consecutiveZeroTarget = 3  # number of consecutive zero checks before exit
consecutiveZeroCount = 0  # current count of consecutive zeros
start_time = time.time()

while True:
    # Check the contents of both opened and closed requests arrays using injected javascript
    opened_requests = driver.execute_script("return window.openedRequests;"); 
    closed_requests = driver.execute_script("return window.closedRequests;");
    print(f"Opened requests: {opened_requests.keys()}")
    print(f"Closed requests: {closed_requests.keys()}")

    if opened_requests.keys() == closed_requests.keys(): # match object
        consecutiveZeroCount += 1
        print(f"Consecutive zero count: {consecutiveZeroCount}")
        if consecutiveZeroCount >= consecutiveZeroTarget:
            break

    if time.time() - start_time > timeout:
        print("Timed out waiting for all requests to complete.")
        break
    
    time.sleep(0.5)  # rerun check every interval 

# finally, close the browser 
driver.quit()
```

## Closing thoughts
Cypress and Playwright are very powerful. You can leverage some of what makes them powerful and port them over to Selenium 3 or 4. How and if you use these tools is up to you, but I have found it very useful to make the tests of my projects more robust by having more granular control of network events (by whatever method is appropriate). Ultimately, whether these methods are useful to you depends on the tradeoffs you are willing or able to make in order to vouchsafe the shippable quality of whatever's under test in a responsible and accurate way. 