---
title: "Couchdb Slowness Gotcha"
date: "2019-11-04"
categories: 
  - "development"
  - "web-development"
tags: 
  - "couchdb"
  - "http"
  - "rest"
  - "sse"
  - "web-app"
---

I was recently using Couchdb with a VueJS frontend and found that as my web app became more complicated, the RESTful responses became gradually really slow to the point of being almost unresponsive.

I am using Server Sent Event subscribers to use the Couchdb changes API for multiple databases. Each change to a specific database triggers a message which I am subscribing too. I use these messages a push notifications to inform me when i need to go and perform a GET and get the updated data.

I found the following problems as my app grew:

- Unresponsive requests
- slowness
- multiple instances of the web app would instantly result in slowness

I found that if i had one instance of the web app in a normal browser and another instance in a private window, then the web app would continue to be responsive. This indicated to me that perhaps there was some problem with sessions/cache or that each connection was using a shared pool of memory that would quickly run out.

I eventually found that the web browser i was using (firefox) has a maximum of 6 connetions per hostname http://www.browserscope.org/?category=network&v=top

![](/images/browser-comparison.png)

This was just a development environment, so i simply just created some additional entries in my hosts file. I had dummy entries like localhost1, localhost2 all resolving to my loopback address.

I then changed each of my subscribers to use a different hostname and found that my webapp became full responsive again.

However, i suppose this means that any more than 6 instances of the web app open in a session will result in the same issue.

Perhaps i should refactor my code...
