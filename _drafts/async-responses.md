---
layout: post
title: "Dissecting Asynchronous Responses in Servlet based Java Applications"
date: 2025-09-24 11:41:00 +1100
ref: asyncjava
comments: true
categories: java async reactive
description: Behind the scenes of asynchronous APIs in Java
---

On a daily basis, if you are working on a Java based application, chances are high that your application provides some REST APIs built with some framework that provides abstraction on top of Servlets. Although Servlet based applications are present still today, it is easy to get lost in the terminology especially when it comes to async processing and non-blocking I/O. Driven by curiosity, I have decided to take a deep dive on what is behind asynchronous responses in Servlet based Java applications down to the syscall level and try to clear up some misconceptions that I believe are still present in the community.


## What you will find in this post

After my little adventure on this topic, I was able to better understand how Tomcat handles ~10,000 concurrent connections with a smaller number of threads using sockets with non-blocking I/O and how asynchronous processing works under the hood at syscall level. In practical terms, when using Spring Web, this means a better understanding of what happens when a controller method returns `DeferredResult`, `WebAsyncTask` or `Callable`. 

Beyond the practical implications, I think that it is just cool to get a peek on the engineering behind the scenes of a technology that is widely used in the industry. It is just beautiful to see how my humble controller interacts with Tomcat inner workings and how that is handled by the OS (in my case Linux).

![Async room]({{ "/assets/async/asyncnonblocking.png" | absolute_url }})

## A bit of history first

One of the things I like about software engineering is the history behind the technologies we use. What led to the development of what I use today? How did we get here? What were the decisions made along the way? What were the problems that motivated these decisions? This is kind of archeology of software engineering, and to me this is as cool as reading and writing code.

To understand the context around how async processing came up in Servlets, I travelled back in time around 2006. The final spec of Servlet 3.0 was released in December 2009 as part of Java EE 6, but the discussions and proposals started years before that. That was the time when people were excited about Web 2.0 (a term popularised in 2004 according to some sources referenced on [Wikipedia](https://en.wikipedia.org/wiki/Web_2.0)). 

Some of the technologies used to create more interactive and more dynamic web applications were AJAX (Asynchronous JavaScript and XML) and Comet (_a style of event-driven, server-push data streaming_ which the term was coined by Alex Russell in a post published in 2006 called [Commet: Low Latency Data for the Browser](https://infrequently.org/2006/03/comet-low-latency-data-for-the-browser/)). 

All of that cool stuff created a new set of challenges for web servers. One single client could open multiple connections to the server, and multiple requests could happen concurrently, increasing the load on the server and at that time, saturating threads. A better support and standardised way for handling slow processing and long-lived connections was needed. To start overcoming these challenges, one of the proposed changes for the Servlet 3.0 was suggested as "Async and Comet support" (see [JSR-315](https://jcp.org/en/jsr/detail?id=315)).

Non-blocking I/O support was also added but was improved later in Servlet 3.1 allowing for non-blocking reads and writes of request and response respectively. That wouldn't yet be a full non-blocking solution as we know today with e.g. Spring WebFlux, but in any case was still a step forward. To me looking back at how things evolved, is fascinating and it was supper cool reading old mailing list discussions, old blog posts accessible via Web Archive and so on.

