---
title:  "Queries in a Pinch"
layout: single
categories: ["Devlog", "Lua", "Communicator"]
tags: [backend, database, cassandra, lua, threading, coroutines, async]
---

# Queries in a Pinch

This post is about a code optimization that speeds up response times by changing how application workers wait for query execution. It discusses why we chose this optimization, what were the available options for a solution, what we chose and how we refactored our code - including some samples. That sounds quite in-depth and technical, and it is. I provided additional links to encase more information (there is only so much one post can discuss) should you decide to go further down the rabbit hole.

Hope you enjoy!

## More Equal Than Others
Database queries are not created equal - some are short and simple, some are long, arduous and complicated. Some take an insignificant amount of time and some make our DBA team cry at night. Some are just unlucky and need to wait for some DNS shenanigans, a lazy network card or some mumbo jumbo about [tombstones in SSTables](https://medium.com/walmartglobaltech/tombstones-in-apache-cassandra-d0a068a72dcc). All of them take _some_ amount of time.

While a query is executing, an application worker is waiting.

A waiting worker is a waiting customer.

A waiting customer might prefer a quicker API.

If a request takes enough time, customers can decide to search for a faster solution.

A few milliseconds can end up costing a company millions of dollars.

A waiting worker is also a waste - computers are generally quite fast these days. Waiting for a _millisecond_ can mean _millions_ of wasted instruction cycles on a modern processor, and query wait times can add up _really, really_ fast. We needed to optimize this wait time, and we needed it yesterday.

## Frankly, my darling, I don't give a _query_
One way we made [Cassandra Communicator](https://moo64c.github.io/articles/2021/08/15/Cassandra-A-Scale-y-Story/) optimize queries is to perform all **write** queries _without replying to the worker_. Application workers send the `insert` or `update` query to the Communicator, and go on their merry way. In most cases, they could not care less if the write query actually succeeded - and for good reason: would they do much more than **write** something to the log and send a metric if the query failed? Communicator can handle that. Most **Write** queries would not block our workers - they do not and should not care about the result. Caring equals waiting on a query before doing something else. In cases when that caring is needed the option exists.

But you always care when reading from the database. Our workers will always wait on **_read_** queries.

**Read** queries produce data, and the worker needs to use the data for something - possibly logic, additional queries or to send it elsewhere. Maybe just verify that it exists. We can not avoid waiting for the query before doing something with the data before we could carry on.

Or is there?

## In Threads We Trust
Asynchronous (aka async) queries allow for a lot of flexibility when optimizing. One way to enjoy the async performance benefit for read queries is to add a [thread](https://stackoverflow.com/a/5201906) per query to wait on its reply. Threading will work well under the assumption at a given point in the code that requires some data from the database, there is a following piece of code that does not require that data at all. Simply: put the task in some thread's capable hands to fetch and process the data, and the worker itself (main thread) can do some other task in the meantime - such as starting more queries. The limit on concurrent threads is how much the processor can handle - running thousands of them at the same time is theoretically possible. When you run out of code to run without the queried data, the main worker thread will have to block until all query threads finish (aka _join_ back to the main thread).

![Sending queries into their own threads and joining the main thread later.](../assets/images/2022-08-20-Query%20Grouping/threading.png)

Adapting existing code, which means splitting the queries into threads, requires more than bit of work. Most of it is finding queries that fit this use case in the code and where they need to _join_ back to the main thread. This makes the optimization come with a hefty cost of complexity in the code. Used incorrectly, the worker can actually end up waiting more - it is up to the developer to find the correct use cases. This makes threads an optimal choice for very specific use cases - such as code that draws a lot of different data sets completely foreign to one another for different uses.

Not only does the threading solution has a chance of not optimizing the worker run time, threads add a not-so-hidden cost of [_context switching_](https://en.wikipedia.org/wiki/Context_switch), which might eliminate any potential performance benefit. Every time you add a thread the eternal headache of locking mechanisms is right around the corner. This is why - as a generic solution - threading did not fit well. More importantly our codebase **does not allow for threading** at all. Everything must happen in the main worker thread or in some other process (like Communicator handling _write_ queries by itself).

Luckily there is more than one solution to optimizing query wait time. Let's go back to the drawing board.

## Come Together, Right Now
Client code for Communicator was implemented with the existing code in mind, meaning every time the worker sends a query to Communicator it blocks until the query completes. Communicator handles itself async queries directly to Cassandra; it made sense allow the worker to send a **_group_** of queries instead of one. From the worker perspective all the queries would run at the same time. Grouped queries would only take the group's worst (longest) query's time to execute - much better than the sum of time of each individual query. There is a certain overhead cost for grouping up (memory wise), also causes sending larger datasets over local TCP sockets - but it is a very cheap trade off.

![The async mechanism reduces worker wait time by waiting for as many queries as possible at once.](../assets/images/2022-08-20-Query%20Grouping/Sync%20V%20Async.png)

Creating queries in groups sounds like a hassle, like the one described for threading above. What about the engineering cost?

In theory, grouping queries might take prior knowledge of which queries are required in some following logic: queries come first in some prefetch of all data, and the logic comes later. That makes for a very rigid flow of execution, adds complexity and prone to many issues - such as following conditional queries cannot be optimized at all. It is a huge headache to rewrite or refactor our existing code (which was not built with async in mind) to fit this prefetch mentality, and we had no intention of rewriting anything on that scale.

Therefore, the question is: How can we group queries without entering a realm of painful rewrites?

## Don't _thread_ on me
Luckily, threads are not the only answer to doing several tasks in "parallel". A lesser known concept called [Coroutines](https://en.wikipedia.org/wiki/Coroutine) allows single-thread "threading". Hopefully not diving too deeply to the technicalities, the idea is to let functions suspend while other functions do their thing for a bit before resuming the original function. `Yielding` inside a running coroutine allows other coroutines to `resume`, which in turn `yield` again (or end) and let the other coroutines `resume`. Explicitly - coroutines allow the worker to jump between tasks, resuming any task in the same point it left it.

 It does sound a bit like threading, but it all happens in a single thread. Under the hood threads actually behave in a manner very close to that description, but the (kernel)[https://www.redhat.com/en/topics/linux/what-is-the-linux-kernel] handles all the _yielding_ and _resuming_ (and a lot more). Coroutines expose this functionality directly in user space (application side) and takes this logic off the operating system. [Lua.org](https://www.lua.org) has some [great usage examples](http://www.lua.org/pil/9.4.html) for coroutines for a more technical perspective (and how it is used in Lua). [This talk](https://www.youtube.com/watch?v=Hi6ICEVVRiw) by Kevlin Henney might be easier to digest than a bunch of Lua code.

Coroutines can be great when waiting for input/output (I/O, i.e. reading from a socket), or when several pieces of code need to reach a similar point before running a costly action (i.e. executing a query). While the socket is waiting for more data in one coroutine, another coroutine can move on and do something else. Since I/O is handled by the kernel, there is no need to babysit it in user space (in the application worker). Coroutines do not require context switching and the kernel is not involved meaning a significant performance advantage over threads.

> Discussing slightly more popular languages than Lua, coroutines were a feature added to Python 3.5 (before that Python had _generators_, which are slightly more complex). To use coroutines define a function as `async` - and it will return a `promise` object when called. To get a response, just `await` on the promise object. This allows the runtime to do other things in the meantime, and [when used with the right tool](https://docs.python.org/3/library/asyncio-task.html#coroutine) it can run a lot of networking stuff simultaneously; I will discuss a pretty cool use case (in another service) in a future post.

Using coroutines has some other terrific property: any piece of code can become a coroutine. Wrap existing code in a function, and it could run concurrently to other piece of code - just add a `yield` in the proper place. Coroutines never require a (mutex)[https://stackoverflow.com/a/34556] on any resources since they do not run in some randomized parallel context like threads. No need to rewrite a lot of ancient database models or adjust the flow of our code; there needs to be minimal intervention in existing code to make it able to get a bunch of queries into a group and execute it. There is also no need to rewrite a bunch of manual code for every use case; a general use class can be made.

Combining coroutines with grouped queries can reduce waiting, with little engineering hassle. Wrapping an existing block of code in a function and letting some mechanism handle when to `resume` or `yield` makes a puny engineering cost - after creating some library to handle it. Each query and its following logic can be kept pretty much the same; we would only need to verify the independence of grouped queries and their following logic from one another.

![Grouped queries with coroutines must wait work to finish before starting the query execution.](../assets/images/2022-08-20-Query%20Grouping/normal_vs_threading_vs_grouped.png)

As the diagram above shows, grouping with coroutines might be worse than the threading example. In most use cases, (including ours) worker code will run in **_nanoseconds_**, but queries usually take **_milliseconds_** - meaning the worker parts (orange in the diagram) are several _orders of magnitude_ of times smaller than any query (red or blue) wait parts. In practice, grouping allows for similar or even better performance than threading in _most_ cases.
_(Disproving this last statement is left as an exercise for the reader)._

## Making Queries Great Again
A new `query_grouping` interface was built to wrap everything discussed here together. It turns the functions containing the queries to coroutines, `yields` when a query is sent during a `group:run()` call and manages the coroutines to `resume` when their query completes. Instead of sending the query immediately, it groups them together, sending it once all coroutines have `yielded`. With minimal additional engineering effort, `query_grouping` can shave several precious _milliseconds_ from any read query. In a common API call it can translate to dozens of milliseconds. Dozens!

In practice, we saw an overall improvement in performance after implementing Query Grouping. Currently, it is only implemented for Cassandra, but there are plans to implement it for MySQL, where the benefits would be much more noticeable in any request. `query_grouping` also forces developers to name the groups, allowing us to expose configuration to turn a specific query group back to "non-grouped" flow if a problem is detected.

## Fresh Samples - How does it look like?
Let's say we have the functions `f1`,`f2` and `f3` (which do not affect each other) each of which contain two queries and do something with their result. Let's say they all run one after another:
```lua
  f1()
  f2()
  f3()
```
In the original code each function would run, query the database twice - making the worker wait on each request - and compute something with the result. The fully synchronous case is the worst performance wise, and there would be **_six_** waiting periods, all adding up.

As the functions and their result do not affect one another, they can all run in one query group. To do so we just need to create a group, add the functions, and run it:
```lua
  local group = query_grouping.new_routines_group("sample")

  group:add(f1)
  group:add(f2)
  group:add(f3)

  group:run()
```
That's it; you add callbacks that turn automatically into coroutines, and `group:run()` blocks until everything is done. Assuming each function have two queries, six queries will execute using just two groups of queries, Blocking **twice** instead of **six** times, with the time cost of the worst query in each group. We could do much better with `query_grouping` in this case, but this is already a major improvement.

#### What if `f3` does not even query the database?
If a function does not query the database, it will never reach a `yield` call, meaning it will run until it is finished. This marks the coroutine as `dead`. We can only move on from `group:run()` after all coroutines `dead`. We would still block twice for two different groups, but there is still a performance boost - the worst queries of two groups is less than the sum of the contained four queries' time.

### The Crème De La Crème Part
#### What if `f1` has two queries (`f1_query1`, `f1_query2`) and `f2`,`f3` have only one each?
Let's say `f1` looks something like this:
```lua
function f1()
  local result1 = f1_query1()
  local result2 = f1_query2()

  do_something(result1, result2)
end
```
In that case, we block twice for two groups: one group run completes the queries for `f2` and `f3` and the query in `f1_query1`, but `f1_query2` will require an additional group (and block again). `f2` and `f3` will resume after the first run and reach their end (since there are no more queries, they will not `yield`). The second group would only execute `f1_query2`, and that is wasteful.

Assuming the queries in `f1` are not co-dependent (`f1_query2` does not need anything from `f1_query2`), we can do better: create a _nested_ query group inside `f1`. Yes, you can create **more groups** inside a running group! This allows `query_grouping` to flatten any _nested_ queries to run in one group, meaning waiting once - regardless of nesting depth and amount of queries:

```lua
function f1()
  local result1, result2
  local group = query_grouping.new_routines_group("nested1")

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
Since the function `f1` is already added to `"sample"` group, accessing `f1` means we are already running inside the `"sample"` group. Calling `group:run()` for the `"nested1"` group causes a nifty hand off in control flow behind the scenes, continuing the `"sample"` group run, which now controls a _child_ group `"nested1"`. The `"sample"` group actually behaves as if the coroutines in `f1` is its own, allowing it to share *one actual grouped request* of queries to Communicator.

All this means that *executing all four* queries would block only once, essentially reducing the wait time to the minimum of the longest query in the group. There is no limit to nesting groups vertically or horizontally - `f1_query1` could have additional nesting if there were more queries inside it, and `f2` or `f3` could also nest its own groups and subgroups. You can have a group run right after a different one. You can stash callbacks to run whenever you would like, and nobody could stop you.

The nested group support is what makes a big difference in the engineering overhead. You never have to check if you are in a flow that already uses `query_grouping` as the interface can adjust to whatever you throw at it. The unit test code for this is absolute bananas - tests are generating entire call trees with multiple layers of coroutines that do X amount of queries and nest groups in all directions.

The only obvious limits to use `query_grouping` is that the callbacks added do not directly depend on one another. Designed carefully, you could even have some results or state from one callback used in a different one, using the queries' `yields` as a guarantee of execution order. The coroutine order of execution in `query_grouping` is completely predictable meaning simplicity in the code.

> As mentioned above, using threads would add a lot more engineering overhead; in theory, you could build a similar interface that creates threads and waits on them to join. This would require additional work since you would end up sharing _state_ between threads, which run in an unpredictable order - breaking anything that is not "thread-safe". For example, starting a [Redis pipeline](https://redis.io/docs/manual/pipelining/) during a one thread would either require an additional connection to Redis or to use a lock to verify we are not in pipeline mode when we should not be in some other thread. Once you need to delve into the underlying implementation details in every use case, you can not "just" add the interface as we can with coroutines.

## It's Time to Stop
The Query Grouping implementation suggests that we do not necessarily need to group the queries - at the `group:add(...)` stage, the request can go to Communicator and start doing its thing, shaving additional precious time and actually matching the performance diagram above for threads (and the `run` can be a literal `await` call). You can send the request, but it would require a rewrite of the Communicator client code on our side. Rewrite seems unlikely considering the potential benefits are slim compared to the benefit we can attain from simply sending everything as a group. The [80-20 rule](https://en.wikipedia.org/wiki/Pareto_principle) applies here; for an additional "20%" performance benefit the engineering cost would be quite high and prone to bugs.

Thank you for reading!

## Credits
Query Grouping was created, designed and supported by Nir Nahum, Adi Meiman and Amir Arbel from Pinpoint R&D Team.
