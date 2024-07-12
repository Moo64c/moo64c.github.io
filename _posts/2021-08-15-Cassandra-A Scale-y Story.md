---
title:  "Cassandra: A Scale-y Story"
layout: single
categories: ["Devlog", "Lua", "Communicator", "Lightly technical"]
tags: [backend, database, cassandra, epoll, lua, luajit, driver, threading]
excerpt: Freeing database resources by reducing the amount of active connections - with a special service.
---

## Whose Scale is This Anyway
A product's scale - size of service, supported traffic and distribution reach - changes over its life cycle. At first, a company will want to push out a product that works. It is safe to assume that scaling up will happen later, when the product has some customers - and it cannot take precedence over getting a product out (or dealing with bugs). It makes sense to let a product generate higher costs, run a bit slower and have fewer features in order to just get it out and sell it. Some compromises are necessary, and in the SaaS world, it can always be fixed later.

> If it will be worth the trouble, somebody - sometime in the future - will take care of it.

## When It's Worth the Trouble
Our Database Administration (DBA) team were upset - every tiny configuration change in our product's cluster sent the Cassandra clusters to a whirlwind of trouble. New connections dropped. Existing connections hang, unable to perform simple queries. Combining their metrics with ours, it was obvious that restarting the application bombards the clusters with thousands of new connections simultaneously. With around three hundred application machines, it made little sense. One connection should be enough for every machine, right? Checking the logs, metrics, and the code revealed that is not the case - every machine in the application cluster creates half a dozen or more connections. This was one problem that was left for a future somebody to fix. Something that was not so critical before, made our clusters and performance degrade to an unusable status.

**Why?**

When our machines start up, a central process (*uWSGI*) creates and controls worker processes. Every worker thread under the central process keeps its memory insulated from the rest. This allows us to use *thread-unsafe* libraries and code - we can not use them in a threaded environment since they keep an internal state, which might break if two workers will access it at the same time. This also forces *every* process to use its own Cassandra connection. A Cassandra connection consists of a cache, a socket connection per Cassandra node (Database Cluster machine), and an additional socket connection that controls it. When you have enough application machines and enough Cassandra nodes, this means hundreds to a couple of thousand connection start up at the same time if the application cluster restarts - for example, after a configuration change, which happens all the time. Cassandra can not handle so much in such a time bracket, and new connections can drop. Many queries will not be wait on connections to complete. Even if the cluster was incrementally gaining machines, at some point the Cassandra side has to handle thousands of connections, slowing everything down.

## The Future is Now
This problem existed before. The effort needed to fix this previously-not-an-issue just did not make sense when you have no more than 100-ish connections working against a small Cassandra cluster. You will see some spikes in graphs when the application clusters restart, but nothing terrible. The product scaled up, making this a primary concern.

A new customer appeared, and the clusters needed to expand to fit the demand. After the clusters settled, DBAs were a bit nervous. After another customer and a bit more scaling up - DBAs made this a central concern for the development group. This "problem for somebody the future" became next week's problem pretty fast. *Future somebody* became now.

## Presenting: Cassandra Communicator
A connection pool was the solution we came up with, named Cassandra Communicator (or just Communicator).

Have too many connections? No problem. Let's just keep a single one in a separate process. Local machine sockets are cheap, fast, reliable, and easy to run. Have every worker open a socket that handles all its Cassandra needs in one stop-shop. Use a simple protocol and send JSON-formatted instructions, and voilà! You got support for any language that can open a local socket. All this - and more - in just one thread and just one connection per machine. A single thread to handle all Cassandra requests from all the worker processes in a machine. Communicator does this in less than 1 millisecond (usually much less) of time added to the database request and response.

![Application machine with Communicator vs. application machine without communicator. Number of connections to Cassandra is constant with Communicator.](https://moo64c.github.io/assets/images/2021-08-15-Cassandra-A%20Scale-y%20Story/2.png)

Reducing the connections meant that restarting twenty nodes cost us twenty disconnects and twenty connects, instead of several hundred. Cassandra clusters are no longer suffering from crippling stress whenever a configuration file changes and the cluster requires a restart. The amount of existing connections dropped by more than 80% - which lead to better performance both in connectivity and queries run time. Most importantly, our DBAs were finally happy.

![Graph of communicator connections and Cassandra Connections. Shows over 3200 communicator clients and 320 Cassandra connections.](https://moo64c.github.io/assets/images/2021-08-15-Cassandra-A%20Scale-y%20Story/1.png)

> The graph above tracks database and Communicator connections from one of our production clusters - Communicator value shown where every client in the graph used to be a database connection.

## Peek Behind the Curtain
Communicator relies on a couple of mechanics: Epoll and the Cassandra C driver.

### Epoll - Unix Magic
[Epoll](https://man7.org/linux/man-pages/man7/epoll.7.html) is a kernel-level function which tracks and reacts to kernel-level events. Epoll can track as many sockets, pipes etc. as you will ever want. You just bind a callback function to an event, sit back, relax and watch the show. Unlike its predecessors (poll and select) it does not require iterating over every watched socket or pipe (which means it runs in O(N)) it knows which event fired without going over the entire list (which runs in O(1)). I used it in Communicator to watch not only multiple sockets but also a pipe that I added to track responses from Cassandra. There is much that you can do with this nifty kernel function - I might return to it in future posts.

### A sync? No, thanks, I already ate
Cassandra's C driver is asynchronous in nature - the responses to every command you send can be "collected" later on. We never really used the asynchronous advantage in our code - the C to Lua wrapper will wait for a response from Cassandra (you can never trust developers to clean up after themselves...) making our calls to Cassandra synchronous. Contrary to this careful approach, the updated wrapper we used in Communicator allows us to control everything Cassandra related that was sprawled across our code base and move it to a centralized spot - meaning we do not have to trust developers with clean-ups. Moreover, asynchronous requests allow us time to handle other requests - which in turn allows us to run Communicator in a single thread.

But wait, who is listening on the socket to and from Cassandra? Well... Communicator is single threaded only in the sense we never added another thread in our code; however the Cassandra C driver loves some extra threads. Usually there is at least one thread per Cassandra node in the cluster. These listen on the sockets for data from Cassandra. We cannot respond directly from the driver thread due to how Lua environments work. When a response arrives, the callback running in the thread writes a single byte to a pipe that triggers the Communicator process (the active Lua environment) to collect the data.

Another advantage to controlling everything Cassandra related inside Communicator are things such as the prepared statement mechanism. Creating a prepared statement saves a query on the Cassandra side - preparing the query to run more efficiently, assuming it will run more than once. These improve Cassandra performance and is almost mandatory when running in a large scale. However, creating and reusing these statements caused a massive spaghetti situation across our application code in multiple features. There might be a dozen different places that cache these statements on the application side. Having a mechanism to control it in a separate process allows us to clean much of our code. I decided to push this into Communicator - even though it was not a part of the task, assuming (correctly) nobody would object.

Cassandra is full of these tiny mechanisms - prepared statements, consistency levels, retry policies, speculative execution to name a few. It makes little sense to have them sprawled across our different projects if they can be handled in a single location that can make decisions based on some compact, centralized logic. Communicator allows us to have it in a single location.

## There is no place like 127.0.0.1
Every machine in the cluster has its own Communicator process these days (inside the same machine as the application) which improves latency immensely, makes sense for thread scheduling, and does not affect one worker more or less. It makes sense that each machine has its own connection to the database, similar to what you would have if the all the workers would use the same Cassandra connection.

But can we get even more performance?

## Optimization Prime
Optimizations are a tricky thing. You can almost always optimize more. Will it be worth it to invest more time and effort, to introduce more dependencies or limitations? When Cassandra was first introduced to our application the multiple connections issue was known, but it was deferred to somebody in the future. It was not worth it at the time. As with any project, sometimes you just have to stop when things are working great, and you are ready to go, even when there is an obvious fault that will not scale well.

![Application response time before and after Communicator. Percentage of requests with response time over 0.5 second drops from 15% to 2% with Communicator](https://moo64c.github.io/assets/images/2021-08-15-Cassandra-A%20Scale-y%20Story/3.png)

Communicator runs great in production - the database graphs became pretty when it launched - pretty enough for the head of R&D to display in the quarterly company talk. I speculate we managed to get about 80% of the performance there was to gain by creating it. But what about the other possible 20% optimization, just sitting there all forgotten?

Well, it is a matter of balance.

What we have left to gain here is by further reducing stress on the Cassandra side will probably come with a performance cost to the application side - such as moving Communicator out of the machines themselves to separate machines, maybe one or two per cluster. You might have a single database connection for a dozen machines! On one side, fewer connections to the database itself; on the other side - the application will have to jump through more hoops to send requests and receive responses, making that "less than 1 millisecond" baseline a thing of the past. We halted current Communicator optimizations without taking it out of the application machines, although it was in the original design (and that is why we used TCP sockets and not Unix sockets).

Since application machines are usually much, much cheaper than database capacity, and we have some extra milliseconds to spend in our SLA, it is likely it will be more economical to move Communicator out of the application machines. Since bottom line affects many decisions in a business environment, I believe it will happen.

I guess *somebody in the future* will have to do that.

Thanks for reading!

Amir

This article will be featured on the [Trusteer Engineering](https://w3.ibm.com/w3publisher/trusteer-engineering/blog) (accessible to IBM employees) and [my personal GitHub](https://moo64c.github.io/) blogs.
