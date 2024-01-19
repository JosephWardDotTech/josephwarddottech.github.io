---
layout: page
title: Network Synchronisation Code Examples
description: >
  Code examples from Test Talk Wales meetup presentation.
hide_description: true
menu: false
---

Thanks for attending the talk!

Here are the code examples we walked through and supporting information.

## How Playwright Does It - Chrome DevTools Protocol (CDP)

```python 
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.common.desired_capabilities import DesiredCapabilities
import time
import json

# your path to chromedriver
chrome_driver_path = './chromedriver' 

# enable Chrome performance logging for granular network event information (using CDP)
capabilities['goog:loggingPrefs'] = {'performance': 'ALL'}
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

The Chrome DevTools Protocol (CDP) allows for very granular control over network activity. Here, we are using it to monitor when requests are sent and when requests are sent. Selenium has [https://www.selenium.dev/documentation/webdriver/bidirectional/chrome_devtools/](various ways) of using CDP, which also allow you to do interesting things like modify outgoing requests and incoming responses, simulate network conditions, etc.

## How Cypress Does It - Network Proxying

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

        # add request to set (checking if it's there) then log it out
        if request_id not in sent_requests:
            sent_requests.add(request_id)
            print(f"Request Sent: {request_details}")

        # add response to set (checking if it's there) then log it out
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

Cypress creates its own proxy to send network requests through. By installing BrowserMob proxy we can mimic this. BrowserMob proxy also has an API for granular network interception that allows us to rewrite requests, responses, and other things just like CDP.

## A Third Way - Monkey Patching

```python
from selenium.webdriver.chrome.service import Service
from selenium import webdriver
import time
import hashlib

# your path to browsermob chromedriver
chrome_driver_path = './chromedriver' 

# Setting up Selenium to use the proxy
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

By injecting JavaScript we can monkey patch the browser's internal methods for sending and receiving responses. This allows us to extend them on the fly with whatever additional code we want. While this is quite powerful, you will notice that the JavaScript should be injected after the page has loaded. Full page navigation will also reset the injected JavaScript, meaning this is typically only useful on single page applications, on features within a web application that won't cause page navigation, etc. 

## Closing thoughts

Cypress and Playwright are very powerful. You can leverage some of what makes them powerful and port them over to Selenium 3 or 4. How and if you use these tools is up to you, but I have found it very useful to make the tests of my projects more robust by having more granular control of network events (by whatever method is appropriate). Ultimately whether these methods are useful to you all depends on the tradeoffs you are willing or able to make in order to vouchsafe the shippable quality of whatever's under test in a responsible and accurate way. 