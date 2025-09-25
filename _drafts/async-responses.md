---
layout: post
title: "Dissecting async and non-blocking processing"
date: 2025-09-24 11:41:00 +1100
ref: asyncjava
comments: true
categories: java async reactive
description: Behind the scenes of asynchronous APIs in Java
---

This post is a result of a deep dive driven by curiosity and the belief that understanding some levels deeper of any abstracted building block, can leverage better decisions when designing systems. In this case I wanted to go as deep as to syscalls level which was a delightful experience that I hope others can experience through this post.

![Async room]({{ "/assets/async/asyncnonblocking.png" | absolute_url }})

## Where it all started

It all started when I was reading a white paper on Reactive Programming vs Reactive Systems. When using reactive programming, we often use non-blocking and asynchronous mechanisms provided by reactive libraries and tooling. Whether using Project Reactor or RxJava with Netty, or using NodeJS, the terminology is usually confusing with asynchronous and non-blocking being used interchangeably. Then I decided to dissect how asynchronous processing is supported in Java, starting with Servlets. This is what led me to this journey of understanding async and non-blocking processing under the hood.

## Motivation

One may think, why bother about async support in a Servlet based application? It is fair that people ask: are you still writing Servlets? Well, I am not actually writing Servlets, but I use Spring Web which is built on top of the Servlet API. As you may know, that can be achieved in Spring Web simply by returning e.g. a `DeferredResult`, `WebAsyncTask` or `Callable` from a controller method ([Spring asynchronous requests](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html)). Long story short, yes it is still relevant to understand how async processing works at Servlet level, although I am going even further with syscalls.

Back to reactive programming, a pedant developer may still argue: by 2025 you should just use Spring WebFlux. Well, I am sorry to disappoint you if you are this pedant developer. The reason being is that as everything else in software engineering, it depends. If your application is tied up with blocking calls to databases or other services, going reactive may very likely not bring you any benefits. Quite the opposite, it can actually make things worse. There is much more to consider when deciding between the two options, but that is not the focus of this post.

## A bit of history first

async only was still an immature solution.

around 2007, people were discussing the proposal of adding async support to Servlets, and Greg Wilkins [link](https://webtide.com/jsr-315-servlet-3-0-specification-part-i/?utm_source=chatgpt.com) (the creator of Jetty) was one of the main contributors to these discussions. He had suggestions based on Jetty continuations and how they could be used to implement async processing in Servlets, however, the spec took a different direction with the introduction of Request.startAsync().

circa end of 2012, there were many discussions at [users at servlet-spec.java.net](https://download.oracle.com/javaee-archive/servlet-spec.java.net/users/2012/11/0294.html) mailing list about the relationship between the Non-Blocking API of 3.1 and the asynchronous processing of Servlet 3.0.


## Async vs non-blocking

As I previously mentioned, these terms are often used interchangeably, but they are not the same thing. In hope of clarifying the distinction, here is a simple breakdown of the two concepts:

* __Asynchronous__: This is about transferring the execution of a task to another thread or process (or even a service). The caller thread in this case, can't wait for the task to complete and thinking about a REST API, the caller transfer the execution of the request and returns from the call. The response continues to be processed by another thread, and in the case of Servlets, the response is written back to the same connection that was used to make the request.
* __Non-blocking__: This is about not blocking the caller thread completely freeing it to do other things while indirectly waiting for a task to complete. When the I/O operation is done, the thread can be notified (usually via a callback or an event) and can then handle the result of the I/O operation.

## An Async request example

Using strace in order to trace system calls, I was able to see what happens when an async request is made to a REST endpoint. The example below shows a simple Servlet application running with embedded Tomcat.


```
strace -ttt -f -o trace.log -e \
trace=network,desc -s 2000 \
java -jar target/asyncservlet-1.0-SNAPSHOT-shaded.jar
```

Summary of the output

```
144523 5438.555 accept(8,  <unfinished ...>
144523 5440.167 <... accept resumed>{sa_family=AF_INET6, sin6_port=htons(59396), ...), sin6_scope_id=0}, [28]) = 11
144523 5440.175 fcntl(11, F_GETFL) = 0x2 (flags O_RDWR)
144523 5440.175 fcntl(11, F_SETFL, O_RDWR|O_NONBLOCK) = 0
144523 5440.177 accept(8, {sa_family=AF_INET6, sin6_port=htons(59400), ...), sin6_scope_id=0}, [28]) = 12
144522 5440.178 epoll_ctl(9, EPOLL_CTL_ADD, 11, {events=EPOLLIN, data={u32=11, u64=126985003073547}}) = 0
144522 5440.178 epoll_ctl(9, EPOLL_CTL_DEL, 11, 0x737e05ffe514) = 0
144512 5440.201 read(11, "GET /hello HTTP/1.1\r\nHost: localhost:8080\r\nConnection: keep-alive...", 8192) = 695
144512 5440.218 write(1, "Processing in thread: http-nio-8080-exec-1\n", 43) = 43
144513 5440.221 write(1, "Processing async in thread: http-nio-8080-exec-2\n", 49) = 49
144513 5444.231 write(11, "HTTP/1.1 200 ...", 136) = 136
144522 5444.232 epoll_ctl(9, EPOLL_CTL_ADD, 11, {events=EPOLLIN, data={u32=11, u64=11}}) = 0
```

The flow explained

1. Accept is initiated by thread 144523 in order to accept new connections.
2. The accept call is resumed, and a new socket file descriptor (fd 11) is created for the incoming connection.
3. The socket is set to non-blocking mode using fcntl.
4. Poller thread 144522 adds the new socket (fd 11) to the epoll instance to monitor it for incoming data.
5. The HTTP request is read from the socket (fd 11) by the worker thread.
6. Removes the socket (fd 11) from the epoll instance as it is being processed.
7. The request is then processed by thread 144513, which starts processing the request asynchronously in another thread.
8. The response is written back to the socket (fd 11) by the worker thread number 144513.
9. The epoll instance is updated again to monitor the socket (fd 11) for incoming data.

The conclusion of this flow is that the request is accepted and processed asynchronously, with the use of multiple threads and non-blocking I/O to handle incoming connections and data efficiently. And the interesting part to me is that the response is written back to the same socket file descriptor (fd 11) that was created when the connection was accepted. Why this is interesting to me? Because the Servlet API abstracts these details away giving the impression that request is partially processed immediately with the Servlet not returning anything from the async processing.

# Links

- https://jcp.org/en/jsr/detail?id=315&utm_source=chatgpt.com
- https://webtide.com/jsr-315-servlet-3-0-specification-part-i/?utm_source=chatgpt.com
- https://niels.nu/blog/2016/spring-async-rest


## Resources to explore in order to write this post

- Reactive programming lessons learned by Tomasz Nurkiewicz https://www.youtube.com/watch?v=z0a0N9OgaAA
- Project showing how async support in Servlets can be done
- Here is where the author explain the reasoning behind the project on GH: https://web.archive.org/web/20110415080928/http://nurkiewicz.blogspot.com/2011/03/tenfold-increase-in-server-throughput.html


