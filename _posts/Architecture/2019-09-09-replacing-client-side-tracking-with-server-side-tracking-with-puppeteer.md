---
layout: post
title: "Replacing client-side tracking with server-side tracking with Puppeteer"
author: gregory.guichard
categories: [en, cloud, web]
image: assets/images/Architecture/tracking_server/thumbnail.jpg
---

Tracking is now an integral part of the web.
All websites have scripts that track the actions of its users to improve their experience, understand their behavior and offer them offers that meet their needs.
However, these trackers require resources from client terminals for their execution, and this can slow down navigation and adversely affect the user experience.

## How does a tracker work?

There are many trackers, the best known are the [Facebook pixel](https://fr-fr.facebook.com/business/) or [Google Analytics] (https://analytics.google.com/analytics/ web /).
They make it possible to follow the number of visits, the pages most consulted, the actions of the users or the characteristics of the users (resolution, terminal and navigator used, operating system, etc.).
A tracker is an element added to an HTML page (eg a Javascript script or an image) that will be run on the client's browser to transmit information to a remote server where it will be collected and analyzed.
Trackers have different purposes, some trackers can improve the relevance of content, adapt to the uses of users, improve SEO, etc. This is why it is common to have several trackers on the same site.
The tracking is often accompanied by a cookie which is installed on the client terminal, in order to follow it during its navigation on the site and possibly during its next visits.

![tracking client vs tracking serveur]({{ site.baseurl }}/assets/images/Architecture/tracking_server/client-server-tracking.png)

## What are the advantages of a Back-End tracking?

In addition to its own tracking, it is customary to install partner trackers that will collect data on their own servers to analyze and report consolidated results.
In the case of advanced sites such as Cdiscount, the number of tracking partners can become important.
It becomes interesting to replace all these trackers with a single tracker and make calls to the server side partners.
Advantages are :

- **Data Quality**: Trackers run on the user's device.
    However, some trackers may not run or execute badly and this creates heterogeneity in the data that is reported.
    By running these trackers on the servers instead of the browser, we can control that they run correctly and thus have quality and consistent data.
    It is even possible to replay trackers retrospectively if needed (eg website of unreachable partner)
- **Best customer experience**: Trackers use users' hardware resources, and for smaller devices this can be a significant expense.
    Freeing users from the execution of these trackers allows better fluidity when browsing the site.
    It also reduces the number of cookies installed

## Issues related to Back-End tracking

There are two cases:

- The tracking partner API is public and documented: in this case there is no particular difficulty. Just execute an HTTP request with the customer data.
- The tracking partner API is not known: in this case we must run the tracker on our servers to mimic the behavior of users. This case is more complicated, and it is this one that will interest us here.

![Schéma tracker]({{ site.baseurl }}/assets/images/Architecture/tracking_server/server_side_tracking.png)

To ensure server-side tracking when the partner tracking API is not known, you must:

- Collect the necessary data for partner tracking in a single Cdiscount tracker.
- Run the JavaScript code of the trackers in an environment simulating the user's browser and its context.
    This being agnostic of the implementation, that is to say without knowing the code and therefore the behavior and information expected by the tracker.

Simulating the navigation on several hundred page views per second can be resource-intensive, the chosen solution must be optimized to avoid excessive processing costs.

## How to simulate a browser Back-End side

To simulate a Back-End side browser we used [**Puppeteer**](https://developers.google.com/web/tools/puppeteer).
Puppeteer is a NodeJS framework that provides a high-level API to control [**Chromium**](https://www.chromium.org/Home) through the [**Chrome DevTool Protocol**](https://chromedevtools.github.io/devtools-protocol/).
Puppeteer is a Google project originally used to test the interfaces of its web applications.
This library is very complete, stable, powerful and well maintained.
Puppeteer will allow to execute the tracking scripts in a web browser for which we will have emulated the context of the users.
The life cycle of our Puppeteer application is as follows:

1. Creating a browser (Chromium by default) via Puppeteer
2. Retrieving the collected tracking data (URLs, JavaScript objects _navigator_, _window_, _document_, etc.).
3. Also recover previously saved cookies if this is not the first run of the tracker for this user.
4. Configuring the browser context via the Pupeteer API.
5. Running the trackers. At the end of the run, the tracker sends the collected information to the partner's server by HTTP requests. The tracker will also generate a cookie as needed. This cookie is saved for reuse if the user visits another page.
6. Close the browser and return to step 1.

This makes it possible to correctly simulate user navigation, but it is not optimized.
On the Cdiscount site, there are several hundred page views per second, and there are several trackers per page.
In order to be able to keep the traffic present on the site, we had to optimize our Back-End tracking strategy:

- **Mutualization of trackers**: the same execution context is used for several trackers.
    The retrieval and setting up of the context (ie user variables) on the virtual browser are expensive, which is why several trackers are executed in the same context.

- **Refresh Strategy**: The page is refreshed instead of destroying and rebuilding the browser.
    It also makes it possible to use the browser cache mechanisms (for images, but also for DOM construction).

- **Enrichment of the data beforehand**: the tracking data are received on a Kafka topic.
    Kafka Streams are used to enrich the tracking data with the corresponding tracking cookie if it has already been generated.

- **Query Execution**: Query execution by the browser is time-consuming. The browser must wait for the response of the HTTP request, handle failures and returns before proceeding to the next tracker.
    This slows down the execution of the trackers. This is why the requests to execute are published in a Kafka topic to be executed asynchronously by a dedicated micro-service.

- **Storing cookies in a cache**: if this is the first time a tracker is run, it usually generates a cookie that must be reused at the next execution.
    This makes it possible to distinguish the users and to follow their navigation.
    This tracking cookie is stored in a CouchBase cache (to be recovered at the time of enrichment).

## Proof-of-Concept Architecture

![]({{ site.baseurl }}/assets/images/Architecture/tracking_server/architecture.png)

1. When a user visits a page on Cdiscount.com, a unique tracking script retrieves the necessary information from the tracking partners.
   It will also set a cookie that will identify the user while browsing.
2. The collected data is then sent to a Cdiscount server.
3. A micro-service developed in Java publishes this data in a Kafka topic.
   Each message corresponds to an execution of the tracking script
4. A Kafka Streams application enriches messages with cookies from partner trackers if they already exist for this user in Couchbase.
5. A microservice consumes the enriched messages and uses them to configure browsers and have them execute partner tracking scripts.
   Rather than running HTTP requests to partner sites, these queries are captured and published in a Kafka topic.
   If the tracker generates a cookie, it is stored in Couchbase so that it can be used in step 4.
6. A Java micro-service retrieves the HTTP requests to be executed to the partners in the Kafka topic and executes them reliably.

## Conclusion

We have seen how to deploy server-side tracking.
This type of tracking brings a lot of difficulties and limitations:

- The information that is sent to the partners must be known in advance in order to retrieve it from the customer and to be able to simulate it in our virtual browsers.
- Some trackers have particular behaviors (eg a delay of a few seconds or waiting for an event before sending a request) and can therefore degrade our performance.
- Browser simulation requires CPU time and also maintenance. This entails a higher cost than client-side tracking.

Nevertheless, Back-End tracking makes it possible to unload the user from the execution of the scripts and thus to offer him a better browsing experience.
It also makes it possible to check the coherence and the quality of the data provided to the partner trackers and thus the coherence of the analyzes provided by these partners.
