---
layout: post
title: "Dissecting Asynchronous Responses in Servlet based Java Applications"
date: 2025-09-24 11:41:00 +1100
ref: asyncjava
comments: true
categories: java async reactive
description: Behind the scenes of asynchronous APIs in Java
---

On a daily basis, if you are working on a Java based application, chances are high that your application provides some REST APIs built with some framework that provides abstraction on top of Servlets. Although Servlet based applications are present in many Java based web applications or services, it is easy to get lost in the terminology especially when it comes to async processing and non-blocking I/O. 

Every backend Java developer knows what is a Servlet and I hope, also knows how it works. It is a staple technology in the Java world and as everything else, it is also easy to take it for granted. As with a terminal emulator, which I investigated on a previous post, there is always more than meets the eyes happening under the hood.

Driven by curiosity, I have decided to take a deep dive on what is behind Servlet's asynchronous support down to the syscall level and to try clearing up some misconceptions that I believe are still present in the community.


## What you will find in this post

After my little adventure on this topic, I was able to better understand how Tomcat handles ~10,000 concurrent connections with a smaller number of threads using sockets with non-blocking I/O and how asynchronous processing works under the hood at both Tomcat internals and syscall level. 

On a personal note, I find joy on investigating what is behind simple things we use everyday. Yesterday was about terminal emulator, today is about Servlet, tomorrow may be what is behind NodeJS event loop (spoiler: I've been playing with libuv already - the library that underpins Node's event loop). For the everyday use of Servlets, it should be a boring topic, but finding the little universe behind it is something else, it is fascinating (at least for me).

![Async room]({{ "/assets/async/asyncnonblocking.png" | absolute_url }})

## Some history first

One of the things I like about the deep dives, is that through source codes and low level APIs, chances are high to also find some history behind the thing being investigated. For Servlets in particular, why async support was added circa 2009? What were the problems being tackled? What was happenning around 2009 and before? What were the alternatives to solution given that we can use still today? This is archeology of software engineering, and to me this is as cool as reading and writing code.

To understand the context around how async processing came up in Servlets, I travelled back in time around 2006 where AJAX was the thing and everyone was excited about [Web 2.0](https://en.wikipedia.org/wiki/Web_2.0). Before the final Servlet 3.0 spec bringing the asynchronous support to the backend, many different approaches were taken to achieve the same or similar problems being solved by Servlet 3.0. Some examples are the [Eclipse Jetty project](https://jetty.org/index.html) and Comet (a style of event-driven, server-push data streaming, which the name was coined by Alex Russell in a post published in 2006 called [Commet: Low Latency Data for the Browser](https://infrequently.org/2006/03/comet-low-latency-data-for-the-browser/)). 

### Here comes the challenges

![Thread starvation]({{ "/assets/async/thread-starvation.png" | absolute_url }})

All of that cool stuff available at that time used to create more interactive web applications such as AJAX, came also with a new set of challenges for the backend. One single client could open multiple connections to the server, and multiple requests could happen concurrently, increasing the load on the server. Just imagine how much the load would increase on a page that was supporting 1000 concurrent requests and suddenly with more interactivity, 10 additional requests would be made via AJAX for that same page. That would go beyond what Tomcat could handle at that time leading to thread starvation and unsatisfied customers. 

A better and standardised way of handling slow processing and long-lived connections was clearly needed. To start overcoming these challenges, one of the proposed changes for the Servlet 3.0 was suggested as "Async and Comet support" (see [JSR-315](https://jcp.org/en/jsr/detail?id=315)). The main focus of this proposal was on supporting asynchronous request processing which by itself should help with the thread starvation problem as well as enabling support for Comet style applications.

Non-blocking I/O support was also considered and further improved with Servlet 3.1 allowing for non-blocking reads and writes of request and response respectively. However, that wouldn't yet be the full non-blocking solution that we know today with e.g. Spring WebFlux running with Netty, or what is provided by other stacks such as NodeJs. In case you want to know more history and see alternatives proposed by the author of Jetty, I left some references at the end of this post that you may like.

### Abstractions

When I learnt about the Servlet's async support a long time ago, I didn't actually use it straight away. When I actually had to use the Servlet Async support, I was actually using an abstraction on top of Servlets provided by Spring Web MVC (to keep it shorter I will call it only Spring Web).

One of the questions I had at that time was: if the response is left open and it is processed by another thread other than the worker, how does Tomcat know how to send the request to the right connection (i.e. to the right client)? I know that it is not reasonable to dive deep into internals on every abstraction that I use in a daily-basis. Doing so would make me the most unproductive developer, so as most people do, I accepted that Tomcat just works as expected and moved on. However, the curiosity was still there.

Alright, so now that Spring is staple framework for Java projects, is it still worth understanding the nitty-gritty details of Servlets and Tomcat? I think so. Regardless of using Spring Web or directly using Servlets (although I expect most projects to be using some abstraction on top of Servlets), the principles remain the same and a solid understanding about it helps clearing out common misunderstandings about asynchronous vs non-blocking I/O support.

<div style="border: 1px solid #b3afd3ff; padding: 0.5em; background-color: #edebfaff; margin-bottom: 0.5em">
Spring Web provides <code>DeferredResult</code>, <code>Callable</code> and <code>WebAsyncTask</code> as controller return types for async support. When returning these types, Spring Web uses Servlet's 3.0 async support in order for the response to be returned from a separate thread (it works a bit different from the example in this page since it relies on dispatching mechanisms). It also allows for <a href="https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-async.html#mvc-ann-async-reactive-types">returning reactive types</a>, although they are all adapted to an equivalent async result type (e.g. a <code>DeferredResult</code> for single value results). 
</div>

## Enough with history, time to see some code

To navigate this exploration, I present it all in a top-to-bottom style, following the diagram shown below. Here I start explaining what `HelloServlet` is doing, then start diving into Tomcat's internals down to the operational system calls via APIs and syscalls. In my case, my exploration started with the Servlet while looking at `strace` logs first.

![Layers]({{ "/assets/async/layers-request.png" | absolute_url }})

## HelloServlet

The servlet's purpose here is very simple:
1. Handle a request to `/hello` endpoint
2. Start an asynchronous context 
3. Hand-off the request processing to a new thread named `custom-thread`. That is enough to free the current the Tomcat's worker thread
4. Finally send the response to the client from `custom-thread` -- a task perfomed by BackgroundTask -- completing the request processing.

The complete source code for this simple Servlet project is available as another repository on my GitHub: [asyncservlet](https://github.com/adolfoweloy/asyncservlet). I recommend downloading it in case you want to run some of the commands that I share in this post.

```java
@WebServlet(value="/hello", name="helloServlet", asyncSupported = true)
public class HelloServlet extends HttpServlet {
    private final static String CUSTOM_THREAD_NAME = "custom-thread";

    @Override
    public void service(HttpServletRequest req, HttpServletResponse res) {
        System.out.println("Handling the request in thread: " 
            + Thread.currentThread().getName());

        AsyncContext asyncContext = req.startAsync();
        asyncContext.setTimeout(10_000);

        new Thread(
                new BackgroundTask(asyncContext),
                CUSTOM_THREAD_NAME
        ).start();
    }
}
```

The background task simply sleeps for four seconds and is responsible to write to the response and to complete the asynchronous processing. The following snippet shows the background task.

```java
private record BackgroundTask(AsyncContext asyncContext) implements Runnable {

    @Override
        public void run() {
            try {
                System.out.println("Processing async in thread: "
                        + Thread.currentThread().getName());
                Thread.sleep(4_000);
                var res = (HttpServletResponse) asyncContext.getResponse();
                res.setContentType("application/json");
                writeResponse("Hello, World", 200);
            } catch (InterruptedException e) {
                writeResponse("db error", 500);
            } finally {
                asyncContext.complete();
            }
        }

        private void writeResponse(String message, int status) {
            var res = (HttpServletResponse) asyncContext.getResponse();
            try {
                res.setStatus(status);
                res.getWriter().println(message);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
}
```

### Running the example

If you want to run the code, in order to have a smooth experience I recommend you install SdkMan in case you still don't have it. Most of my Java projects come with a `.sdkmanrc` with the Java version being used for the project and in order to use the right version from the terminal, all that is needed is to run `sdk env` (IntelliJ recognises it straight away picking-up the right JVM for the project -- of course it expects you to have the JVM installed).

Once you have the Java 21 installed, you can simply run the following commands:

```bash
mvn clean package
java -jar target/asyncservlet-1.0-SNAPSHOT-shaded.jar
```

I am also using `wrk` as a benchmarking tool in order to compare asynchronous vs synchronous processing. Here is its [GH project page](https://github.com/wg/wrk). With `wrk` you can send concurrent requests using multiple connections as in the snippet below. On this example I am creating 10 connections and 20 threads with a duration of 10 seconds. This should allow one to see how many requests can be handled within this period of 10 seconds given the conditions set in the parameters.

```bash
wrk --timeout 10s --latency -t20 -c10 -d10s http://localhost:8080/hello
```


## Tomcat internals

First things first, in this blog post I examine two separate aspects of Tomcat: the low-level networking level and HTTP request pipeline. The former manages socket creation, polling and event dispatching while the latter actually process the request by parsing it, applying filters, invoking the Servlet's `service` method and process the response.

Before diving in code or logs, lets examine the networking level starting with the actors that collaborate to accept connections and handle the request.

### The actors behind a request

I think the easiest way to actually see the actors at play, is by actually sending requests to `/hello` endpoint and monitoring Tomtcat's threads by using a tool like [VisualVM](https://visualvm.github.io/) or [jconsole](https://openjdk.org/tools/svc/jconsole/). 

The following image, captured from VisualVM while sending a request `/hello` endpoint, shows the threads that are important for the purposes of this post. 

![Tomcat threads]({{ "/assets/async/selected-threads.png" | absolute_url }})

Straight to the point, the actors are the Acceptor, the Poller, some Workers and the Custom Threads. Here is the breakdown of each one:

- __Acceptor__: accepts new connections from clients and hand the request over to the poller by adding a `PollerEvent` to the `Poller`'s internal queue.
- __Poller__: the poller blocks for one second waiting for new events on the queue, or is "woken-up" in between the next poll iteration. Whenever there is an event to process, the poller offloads the processing to a worker thread, also known from Tomcat's Javadocs and source code as _container thread_. I will keep calling it worker thread.
- http-nio-8080-exec-N: this is actually an instance of Tomcat's __worker__ thread. Tomcat starts with already a pool of ten threads ready to process incoming requests. By default, it can create a maximum of 200 requests.This information can be obtained from Tomcat's documentation, but can also be retrieved from JMX Tomcat's ThreadPool attributes (VisualVM doesn't come with the MBeans tab out-of-the-box, so if you want to see this for yourself, you have to install MBean plugin for VisualVM).
- custom-thread: this is the thread I am creating from within `HelloServlet#service` method. I will call it just, well, __custom thread__.


### The networking level classes

Now to put things in perspective, the actors previously described are all instances of some of the classes depicted in the class diagram __below__. I consider `NioEndpoint` the main class in this diagram because `Poller` and `Acceptor` are all inner classes and they all collaborate to handle the low-level networking work, as well as coordinates the communication via Worker threads. As an attempt to shortly describe what is in this picture, these classes collaborate to accept connections, handle the request to the workers and start the HTTP request pipeline via `SocketProcessor`.

![Tomcat main networking classes]({{ "/assets/async/tomcat-main-classes.png" | absolute_url }})



## TODO: use this to connect the dots
With the class diagram above, now we have a map to guide us through what comes next: the request flow.

1. First off the web app loader will start important things for the container such as the endpoint that according to Tomcat's Javadocs, _provides low-level network I/O_ (see `AbstractProtocol#endpoint`).
2. Then when the endpoint's start method is called it creates and starts both the `Poller` and the `Acceptor` in the same order. 
3. When a request comes in, the `Acceptor` calls the endpoint's method that starts accepting new connections. Once a connection is accepted, it calls `endpoint#setSocketOptions` for the accepted socket which will among others, set the following properties:
    - sets the connection to non-blocking
    - sets the read timeout
    - sets the write timeout
    - sets the keep alive left attribute
    - registers the socket (wrapped in a `SocketWrapper`) against the `Poller`.
4. When the `SocketWrapper` is registered in the `Poller`'s queue, which will check whether that is the first event, in which case the poller will be woken-up via selector starting the request processing. Otherwise, it will just enqueue a new event in case of multiple requests coming in a very short interval.
5. Whenever the `Poller` has an event in the queue to be processed, it will call its method `processKey` (key here is what is returned from a selector -- see more about Java Nio to understand it) which in turn will call the endpoint's `processSocket` method.
6. `processSocket` creates a `SocketProcessor` if one is not present, and submit that to Tomcat's worker thread pool executor.
7. The `SocketProcessor` is also the entry-point for the HTTP request pipeline to start.

## System calls

When I first started exploring how Tomcat handles requests under the hood, I wanted to see how the custom thread was actually sending the response back to the right client. Although the log can be huge, knowing the calls to look for can help one understanding what is going on without having to actually look at Tomcat's source code. 

But how to get system calls' logs? Enters [strace](https://man7.org/linux/man-pages/man1/strace.1.html).

The description from strace's man page is very clear about what it does:

> It intercepts and records the system calls made by a process and the signals a process receives

But reading the second paragraph of the description section is even more exciting and inspiring:

> strace is a useful diagnostic, instructional, and debugging tool. System administrators, diagnosticians, and troubleshooters will find it invaluable for solving problems with programs for which source code is not readily available, as recompilation is not required for tracing. Students, hackers, and the overly-curious will discover that a great deal can be learned about a system and its system calls by tracing even ordinary programs.

Before jumping into the logs, it is worth mentioning that it logs both the processes IDs (PID) as well as thread IDs (TID).

### Starting the service with strace

The two options I know about on how to start tracing the system calls logs are:
- attach strace via PID
- start the application with strace

In my example I will show what I actually used which was the second option:

```bash
strace -ttt -f -o trace.log -e trace=network,desc -s 2000 \
java -jar target/asyncservlet-1.0-SNAPSHOT-shaded.jar
```

Because of the way I started, I quickly sent a request to `/hello` in order to generate the logs and soon after I got the response I interrupted the command above (Ctrl + C). The logs were huge: 24 MB for a couple of seconds. But don't worry, I summarised what was relevant for my purposes and here is what I got (the timestamps were truncated in hope to get each line shorter):

```
144471 5438.553 epoll_create1(EPOLL_CLOEXEC) = 9 
144523 5438.555 accept(8,  <unfinished ...>
144523 5440.167 <... accept resumed>{sa_family=AF_INET6, sin6_port=htons(59396), ...), sin6_scope_id=0}, [28]) = 11
144523 5440.175 fcntl(11, F_GETFL) = 0x2 (flags O_RDWR)
144523 5440.175 fcntl(11, F_SETFL, O_RDWR|O_NONBLOCK) = 0
144523 5440.177 accept(8, {sa_family=AF_INET6, sin6_port=htons(59400), ...), sin6_scope_id=0}, [28]) = 12
144522 5440.178 epoll_ctl(9, EPOLL_CTL_ADD, 11, {events=EPOLLIN, data={u32=11, u64=126985003073547}}) = 0
144522 5440.178 epoll_wait(9, [{events=EPOLLIN, data={u32=11, u64=126985003073547}}], 1024, 0) = 1 
144522 5440.178 epoll_ctl(9, EPOLL_CTL_DEL, 11, 0x737e05ffe514) = 0
144512 5440.201 read(11, "GET /hello HTTP/1.1\r\nHost: localhost:8080\r\nConnection: keep-alive...", 8192) = 695
144512 5440.218 write(1, "Processing in thread: http-nio-8080-exec-1\n", 43) = 43
144513 5440.221 write(1, "Processing async in thread: http-nio-8080-exec-2\n", 49) = 49
144513 5444.231 write(11, "HTTP/1.1 200 ...", 136) = 136
144522 5444.232 epoll_ctl(9, EPOLL_CTL_ADD, 11, {events=EPOLLIN, data={u32=11, u64=11}}) = 0
```

- The first column shows the PID/TIDs
- The second column shows the truncated timestamp
- Then comes the commands

### A prime on the system calls

Before I explain what is actually happening with all these system calls, I think it is worth having an overview of each command used here.

#### The epoll API

The calls `epoll_create1`, `epoll_ctl` and `epoll_wait` are all part of epoll API. According to [epoll's man page](https://linux.die.net/man/7/epoll), epoll is an I/O event notification facility. Depending on the OS being used, Tomcat will use whatever is available. On MacOS the same is achieved with kqueue. In a maybe over simplified idea, the epoll can be use to watch file descriptors (e.g. pipes, sockets and devices are represented as file descriptors in Linux and you can think of them as handlers).

In other words, you give epoll a list of file descriptors to monitor for changes (e.g. the socket is ready for reading or writing) and wait for events to happen. One more thing: `epoll_wait` blocks while waiting for a notification, so here I was able to see that the Servlet's async support is intrinsecaly blocking. However, of course Tomcat don't just sit there forever waiting for notifications. There is actually a loop where it epolls for one second. On the next iteration it may have happened that an event is ready to be processed and that's when the following system call happens (notice that the last argument is 0 -- timeout is set to 0 and it doesn't wait to get the event notification):

```
epoll_wait(9, [{events=EPOLLIN, data={u32=11, u64=xx}}], 1024, 0) = 1 
```

For more about the epoll, I think the blog post "[The method to epoll's madness](https://copyconstruct.medium.com/the-method-to-epolls-madness-d9d2d6378642)" is awesome. I also recommend this nice overview about it from Julia Evans blog, "[Async IO on Linux: select, poll, and epoll](https://jvns.ca/blog/2017/06/03/async-io-on-linux--select--poll--and-epoll/)"

#### fcntl

I will be short on this one because I honestly didn't explore it much, but the bottomline is that it performs operations on open file descriptors. The reason I am mentioning it here is because there is an interesting call in the strace logs:

```
fcntl(11, F_SETFL, O_RDWR|O_NONBLOCK) = 0
```

This command is simply setting the socket's file descriptor property O_NONBLOCK, which means that this file descriptor is non-blocking I/O. Alright so we have non-blocking I/O support, right? Not too fast! That is a property set for the connection only, which means that each time one tries to read from the socket and there is nothing to read, instead of blocking it returns a EAGAIN and the program continues until next attempt to read.


## Connecting the dots

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





really good stating the problem solved by the async processing in Servlets: https://web.archive.org/web/20190315183213/http://www.softwareengineeringsolutions.com/blogs/2010/08/13/asynchronous-servlets-in-servlet-spec-3-0/

> First, was the issue of thread starvation. Each thread consumes server side resources – both memory and CPU resources. As a result, the number of threads within the pool must be constrained to some finite number. Furthermore, if the processing that the thread must perform for a client requires some blocking operation (say waiting on some slow external resource such as a database or web service), that thread is effectively unavailable to the server. In other words, it is possible that all threads in the pool are soon blocked and incoming requests can no longer be effectively handled. This is not just a theoretical problem. With the prevalence of fine grained Ajax-requests, and the adoption of SOA, the load on web servers is on an upward trend.



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


# References
- [Discussion about the relationship between Non-blocking I/O support from Servlet 3.1 and the async processing of Servlet 3.0](https://download.oracle.com/javaee-archive/servlet-spec.java.net/users/2012/11/0294.html)
- [Non-blocking I/O using Servlet 3.1 example by Arun Gupta](https://web.archive.org/web/20131024234453/https://blogs.oracle.com/arungupta/entry/non_blocking_i_o_using)
- [JSR 315 Servlet 3.0 Specification Part I](https://webtide.com/jsr-315-servlet-3-0-specification-part-i/)

# Footnotes
[^1]: https://akka.io/blog/reactive-programming-versus-reactive-systems
[^2]: https://projectreactor.io/docs/core/release/reference/gettingStarted.html


- https://jcp.org/en/jsr/detail?id=315&utm_source=chatgpt.com
- https://webtide.com/jsr-315-servlet-3-0-specification-part-i/?utm_source=chatgpt.com
- https://niels.nu/blog/2016/spring-async-rest
- about comet https://infrequently.org/2006/03/comet-low-latency-data-for-the-browser/

## Resources to explore in order to write this post

- Reactive programming lessons learned by Tomasz Nurkiewicz https://www.youtube.com/watch?v=z0a0N9OgaAA
- Project showing how async support in Servlets can be done
- Here is where the author explain the reasoning behind the project on GH: https://web.archive.org/web/20110415080928/http://nurkiewicz.blogspot.com/2011/03/tenfold-increase-in-server-throughput.html


