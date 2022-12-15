---
title:  "Queries in a Pinch"
layout: single
categories: ["Devlog", "Lua", "Communicator"]
tags: [backend, database, cassandra, lua, threading, coroutines, async]
---

## ...More Equal Than Others
Database queries are not created equal - some are short and simple, some are long, arduous and complicated. Some take an insignificant amount of time and some make our DBA team cry at night. Some are just unlucky and need to wait for some DNS shenanigans, a lazy network card or some mumbo jumbo about [tombstones in SSTables](https://medium.com/walmartglobaltech/tombstones-in-apache-cassandra-d0a068a72dcc). All of them take _some_ amount of time. While the query is executing, an application worker is waiting. A waiting worker is waiting customer. A waiting customer might prefer a quicker API. If a request takes over a few hundred milliseconds, a customer might decide to start scouting for a faster solution. Few milliseconds can cost millions in lost contracts.

A waiting worker is also a waste, computers are generally quite fast - waiting for a millisecond equals around 3 million wasted instruction cycles on a modern CPU. These query wait times add up _really, really_ fast. We needed to optimize this wait time, and we needed it yesterday.

## Frankly, my darling, I don't give a _query_
One way we made [Cassandra Communicator](https://moo64c.github.io/articles/2021/08/15/Cassandra-A-Scale-y-Story/) optimize queries is to perform all **write** queries _without replying to the worker_. Application workers send the `insert` or `update` query to the database, and go on their merry way. They could not care less if the write query actually succeeded - and for good reason: would they do much more than **write** something to the log if the query failed? Communicator can handle that. Most **Write** queries no longer bother our workers. Caring causes waiting on a query before doing something else, and requires adding a flag to the query when sent to Communicator.

But you always wait for **read** queries.

**Read** queries produce rows of data, and the worker needs to use the data for something - possibly logic, additional queries, or to send it elsewhere. Maybe just verify that it exists. There is no avoiding the fact that the **worker needs to wait** for the query to return and do something with it before it can move on.

Or is there?

## When in Doubt, Add a Thread
Asynchronous (aka async) queries allow for a lot of flexibility when optimizing. One way to enjoy the async performance benefit for read queries is to add a thread per query to wait on its reply. Threading will work well under the assumption at a given point in the code that requires some data from the database, there is a following piece of code that does not require that data at all. Simply: put off some task while fetching its data, and we can do some other task in the meantime.

Code that needs the queried data can be moved to a thread, waiting on the query and performing the associated work. This lets the worker (or main thread) do run some other code in the meantime. That code could spawn even more threads for additional queries. When there is no more code to run without the queried data, the main worker thread will have to block until all other threads finish (aka _join_ back to the main thread).

![Sending queries into their own threads and joining the main thread later.](../assets/images/2022-08-20-Query%20Grouping/threading.png)

Splitting the queries into threads requires a bit of work - most of it is finding queries that fit this use case in the code and where they need to _join_ back to the main thread meaning the optimization comes with a hefty cost of complexity. Used incorrectly, the worker can actually end up actually waiting more - it is up to the developer to find the correct use cases. This makes threads an optimal choice for very specific use cases - such as code that draws a lot of different data sets completely foreign to one another for different uses.

Not only does the threading solution has a chance of not optimizing the worker run time, threads add a not-so-hidden cost of [_context switching_](https://en.wikipedia.org/wiki/Context_switch), which might eliminate any potential performance benefit; in addition, every time you add a thread the eternal headache of locking mechanisms is right around the corner. This is why - as a generic solution - threading does not fit well. More importantly our codebase  **does not allow for threading** at all. Everything must happen in the same thread or in some other process (well, like Communicator).

Luckily, there is more than one solution to optimizing query wait time. Lets go back to the drawing board.

## Come Together, Right Now
Our client for Communicator was implemented with the existing synchronous code in mind meaning every time worker sends a query to Communicator it blocks until the query completes. Communicator handles itself async queries directly to Cassandra so it makes sense to have the worker send a group of queries instead of one - from the worker perspective they will all run at the same time. Grouped queries would only takes the group's worst query time to execute, instead of the sum of times for all queries. There is a certain overhead cost for grouping up (memory wise, mostly) and sending larger datasets over local TCP sockets - but it a very worthy tradeoff.

![The async mechanism reduces worker wait time by waiting for as many queries as possible at once.](../assets/images/2022-08-20-Query%20Grouping/Sync%20V%20Async.png)

Creating queries in groups sounds like a hassle, like the one described for threading above. What about engineering cost?

In theory, grouping queries might take prior knowledge of which queries are required in some following logic, and as such - queries come first in some prefetch, and the logic comes later. That makes for a rigid flow of execution, adds complexity and prone to many issues - such as conditional follow up queries cannot be optimized at all. Most importantly it is a huge headache to rewrite or refactor our existing code (which was not built with async in mind) to fit the prefetch flow.

Therefor the question is: Can we group queries without entering a realm of painful rewrites?

## Don't _thread_ on me
Luckily, threads are not the only solution to doing several things in "parallel". A lesser known concept called [Coroutines](https://en.wikipedia.org/wiki/Coroutine) allows single-thread "threading". Without diving too deeply to the technical details, the idea is to let functions suspend while other functions do their thing for a bit before resuming the original function. `Yielding` inside a running coroutine allows other coroutines to `resume`, which in turn `yield` again (or end) and let the other coroutines `resume`. Explicitly - coroutines allow the worker to jump between tasks, resuming any task in the same point it left it.

 It does sound a bit like threading, but it all happens in a single thread. [Lua.org](https://www.lua.org) has some [great usage examples](http://www.lua.org/pil/9.4.html) for a more technical perspective (and how it is used in Lua). However [this talk](https://www.youtube.com/watch?v=Hi6ICEVVRiw) by Kevlin Henney might be easier to digest than a bunch of Lua code.

Coroutines can be great when waiting for I/O (i.e. reading from a socket), or when several pieces of code need to reach a similar point before running a costly action (i.e. executing a query). While the socket is waiting for more data in one coroutine, another coroutine can move on and do something else. Since I/O (such as socket read or write) is handled by the kernel, there is no need to babysit it in user space (in the application worker). Most importantly - coroutines do not require context switching and the kernel is not involved meaning a significant performance advantage over threads.

Using coroutines has some other terrific property: any piece of code can become a coroutine - wrap it up in a function and it could run concurrently to other piece of code, just add a `yield`. Coroutines never require a mutex (lock) on any resources since they do not run in some randomized parallel context like threads. No need to rewrite a whole lot of ancient database models or adjust the flow of our code; there needs to be minimal intervention in existing code to make it able to get a bunch of queries into a group and execute it. There is also no need to rewrite a bunch of code for every use case; a general use class can be made.

> Coroutines were actually a big feature added to Python 3.5 (before that Python had _generators_, which are slightly more complex). To use coroutines simply define a function as `async` - and it will return a `promise` object when called. To get a response, just `await` on this promise. This allows the runtime to do other things in the meantime, and [when used with the right tool](https://docs.python.org/3/library/asyncio-task.html#coroutine) it can run a lot of networking stuff simultaneously. But more on that in a future post about another service.

Combining coroutines with grouped queries can gain much of the performance lost to waiting on queries with little to nothing on the hassle side. Wrapping an existing block of code in a function and letting some mechanism handle when to `resume` or `yield` makes the engineering cost quite tiny after a short effort creating some library to handle it. Each query and its following logic can be kept pretty much the same as we will not need to change anything inside entire blocks of code.

![Grouped queries with coroutines must wait work to finish before starting the query execution.](../assets/images/2022-08-20-Query%20Grouping/normal_vs_threading_vs_grouped.png)

According to the diagram above grouping with coroutines might be worse than the threading example, but in most use cases (including ours) most worker code will run in _nanoseconds_ where any query will take _microseconds_ - meaning the worker (orange) parts are thousands of times smaller than any query (red/blue) parts. In practice, grouping allows for similar or better performance than threading in _most_ cases.
Disproving this last statement is left as an exercise for the reader.

## Making Queries Great Again
Query Grouping is name of the interface we created that wraps everything together. We can turn the functions containing the queries to coroutines, and we can manage the coroutines to resume when their query completes. The only thing left is to force queries to `yield` if they are trying to query in a Query Grouping context, and voila! We can shave several milliseconds from any request involving the database, which is all of them.

We saw an overall improvement in performance after implementing Query Grouping. It is currently only implemented for Cassandra but there are plans to implement it for MySQL, where the benefits would be much much more noticeable in every request. Query Grouping also forces developers to name the groups, allowing us to expose configuration to roll back turn a specific query group back to non-coroutine flow if a problem is detected.

### Fresh Samples
Lets say we have the functions `f1`,`f2` and `f3` (which do not affect each other) each of which contain two queries and do something with their result. Lets say they all run one after another:
```lua
  f1()
  f2()
  f3()
```
In the original code each function would run, query the database - making the worker wait on the request - and compute something with the result. The fully synchronous case is the worst performance wise, and there would be **six** waiting periods.

As the functions and their result do not affect one another, they can all run in one query group. To do so we just need to create a group, add the functions, and run it:
```lua
  local group = query_grouping.new_routines_group("group1")

  group:add(f1)
  group:add(f2)
  group:add(f3)

  group:run()
```
That's it; you add callbacks that turn automatically into coroutines, and `group:run()` blocks until everything is done. Assuming each function have two queries, six queries will execute using just two groups of queries, Blocking **twice** instead of **six** times, with the time cost of the worst query in each group.

#### What if `f3` does not even query the database?
If a function does not query the database, it will never reach a `_yield_` call, meaning it will run until it is finished. This marks the coroutine as `dead`. We can only move on from `group:run()` after all added (and nested) coroutines `dead`. We would still block twice for two different groups, but there is still a performance boost - the worst queries of two groups is less than the sum of the contained four queries' time.

#### But what if `f1` has two queries (`f1_query1`, `f1_query2`) and `f2`,`f3` have only one each?
```lua
function f1()
  local result1 = f1_query1()
  local result2 = f1_query2()

  do_something(result1, result2)
end
```
In that case, we block twice for two groups: one group run completes the queries for `f2` and `f3` and the query in `f1_query1`, but `f1_query2` will require an additional group (and block again). `f2` and `f3` will resume after the first run and reach their end (since there are no more queries, they will not `yield`). The second group would only execute `f1_query2`, and that is a bit wasteful.

Assuming the queries in `f1` are not co-dependent (`f1_query2` does not need anything from `f1_query2`), we can do better: create a _nested_ query group inside of `f1`. Query Grouping can flatten any _nested_ queries to run in parallel to one another - regardless of nesting depth:

```lua
function f1()
  local result1, result2
  local group = query_grouping.new_routines_group("group_f1")

  group:add(function()
    result1 = f1_query1()
  end)
  group:add(function()
    result2 = f1_query2()
  end)

  group:run()

  do_something(result1, result2)
end
```
In this case, executing all four queries would block only once, essentially reducing the wait time to the minimum of the longest query in the group. There is no limit to nesting groups vertically or horizontally - `f1_query1` could have additional nesting if there were more queries inside of it, and `f2` or `f3` could also nest its own groups and subgroups.

This nested group support is what really makes a big difference in the engineering overhead - you never have to check if you are in a flow that already uses Query Grouping as the system will adjust to whatever you throw at it. The unit test code for this is bananas - tests are literally generating an entire call trees with multiple layers of coroutines that do X amount of queries and nest more groups in all directions.

The only obvious limits to use Query Grouping is that the callbacks added do not directly depend on one another - you could share state to some degree and designed carefully - even have some results from one callback used in a different one. There are no guarantees on order of execution inside a Group except the yields between queries, which means you should avoid using a Redis pipeline in one coroutine and letting things jump around.

## Where to Stop
The Query Grouping implementation suggests that we do not necessarily need to group the queries - at the `group:add(...)` stage, the request can go to Communicator and start doing its thing, shaving additional precious time and actually matching the performance diagram above for threads (and the `run` can be a literal `await` call). You can send the request, but it would require a rewrite of the Communicator client code on our side. Rewrite seems unlikely considering the potential benefits are slim compared to the benefit we can attain from simply sending everything as a group. The [80-20 rule](https://en.wikipedia.org/wiki/Pareto_principle) applies here; for an additional "20%" performance benefit the engineering cost would be quite high and prone to bugs.

Thanks for reading!

## Credits
Query Grouping was created, designed and supported by Nir Nahum, Adi Meiman and Amir Arbel from Pinpoint R&D Team.
