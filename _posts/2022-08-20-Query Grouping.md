---
title:  "title tbd"
layout: single
categories: ["Devlog", "Lua", "Communicator"]
tags: [backend, database, cassandra, lua, threading, coroutines, async]
---
<!-- todo: title -->

## ...But Some are More Equal Than Others
Database queries are not created equal. Some are short and simple, some are long, arduous and complicated. Some take a tiny amount of time, some make our DBA team cry at night. Some are just unlucky and need to wait up for some DNS shenanigans or a lazy network card or some [mumbo-jumbo about tombstones in SSTables](https://medium.com/walmartglobaltech/tombstones-in-apache-cassandra-d0a068a72dcc). All of them take _some_ amount of time. While the query is executing, a uWSGI worker is waiting. Database queries are a major overhead for uWSGI workers which adds up _really, really_ fast. Naturally, queries became a favorite target of optimization. One way of optimizing worker execution is optimizing the wait time for queries - reducing the amount of time any worker will wait for queries as much as possible.

## Frankly, my darling, I don't give a _query_
One way the [Cassandra Communicator](https://moo64c.github.io/articles/2021/08/15/Cassandra-A-Scale-y-Story/) optimizes queries - it performs all **write** queries - unless requested otherwise - without replying to the worker. Workers just fire and forget about any `insert` or `update` to the database, and go on their merry ways. They could not care less if the write actually succeeded - and for good reason: would they do much more than **write** something to the log if the write failed? Communicator can handle that. **Write** queries no longer bother our workers.

But **read** queries do.

**Read** queries produce rows of data, and the worker needs to use the data for something - possibly logic, additional queries, or to send it elsewhere. Maybe just verify that it exists. No way to avoid the fact that the worker needs to wait for the query to return and do something with it before it can move on. Or is there? 

<!-- todo: better title -->
## When in Doubt, Add a Thread
One way to enjoy the async benefit for read queries is just more threads. Threading will work well under the assumption at a given point in the code that requires some data from the database, there is a following piece of code that does not require that data at all. This means that whatever needs the database info can get be moved to a thread, that does all that waiting and what not and lets the worker do some other work in the meantime.

![Sending queries into their own threads and joining the main thread later.](https://moo64c.github.io/assets/images/2022-08-20-Query%20Grouping/threading.png)

Splitting the queries into threads can be quite simple - most of the work is just finding queries that fit this use case in the code and where they need to _join_ back to the main thread. The immediate benefit of worker time is acquired, and there's no need to actually group any queries together. The worker might still wait for some queries, so it is up to the developer to find the correct use cases. This could be an optimal choice for code that draws a lot of different data sets completely foreign to one another and does things only on one of them. In a worse-case scenario - a query which everything else has to wait for it - there will be zero benefit (and might be even slower due to threading overhead).

Not only does the threading solution has a good chance of not optimizing the worker run time - working with Lua, especially in the uWSGI context, does not allow for threading. Everything must happen in the same thread or in some other process. Lua was just not built with threading in mind. What else can reduce the worker wait time?

## Come Together, Right Now
Synchronous queries will always take at least the sum of all queries' time. However, grouping queries together and sending them at the same time will only take the time for the worst query in the group to execute and return. There is a certain overhead cost for grouping up and sending larger datasets over network - but it is negligible. This means the more queries that can be grouped together - the less time the worker waits on the database.

![The async mechanism reduces worker wait time by waiting for as many queries as possible at once.](https://moo64c.github.io/assets/images/2022-08-20-Query%20Grouping/Sync%20V%20Async.png)

Grouping queries might take prior knowledge of which queries are required in some following logic, and as such - queries come first in some prefetch, and the logic comes later. Grouping queries in such a way makes for a rigid flow of execution, and is prone to issues - such as conditional follow up queries cannot be optimized at all. Moreover, it is a huge headache to rewrite or refactor existing code (which was not built with async in mind) to fit the prefetch mentality.

Is grouping queries possible without rewriting half of the code?

## Don't _thread_ on me
There is a old concept called (Coroutines)[https://en.wikipedia.org/wiki/Coroutine] that allows single-thread "threading". Without diving too deeply to the technical details, the idea is to let functions suspend while other functions do their thing for a bit before resuming the original function; using a bit more technical terms, _Yielding_ a running coroutine allows other coroutines to _resume_, which in turn yield (or return) and let the other coroutines _resume_. (Lua.org)[https://www.lua.org] has some (great usage examples)[http://www.lua.org/pil/9.4.html] for a more technical perspective. (This talk)[https://www.youtube.com/watch?v=Hi6ICEVVRiw] by Kevlin Henney might be easier to process than a bunch of Lua code.

Coroutines can be great when waiting for I/O (i.e. reading from a socket), or when several pieces of code need to reach a similar point before running a costly action (say, executing a query). While the socket is waiting for more data, the thread can move on and do something else. Coroutines never require a mutex (lock) on any resources since they do not run in parallel. Most importantly - coroutines do not require context switching since the kernel is not involved so they have a significant performance advantage over threads.

Using coroutines has some other terrific property: any piece of code can become a coroutine - wrap it up in a function and it could run concurrently to other piece of code, just add a yield. No need to rewrite a whole lot of ancient database models or adjust the flow of our code; there needs to be minimal intervention in existing code to make it able to _group_ a bunch of queries and to run them in parallel. There is also no need to rewrite a bunch of code for every use case; a general use class can be made.

And naturally we called it Query Grouping.

<!-- todo: better subtitle -->
## Query Grouping
