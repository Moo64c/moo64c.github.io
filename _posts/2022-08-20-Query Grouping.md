---
title:  "Queries in a Pinch"
layout: single
categories: ["Devlog", "Lua", "Communicator"]
tags: [backend, database, cassandra, lua, threading, coroutines, async]
---

# Queries in a Pinch

This post is about a code optimization that speeds up response times by changing how application workers wait for query execution. It discusses why we chose this optimization, what were the available options for a solution, what we chose and how we refactored our code - including some samples. That sounds quite in-depth and technical, and it is, but I kept the code samples to the end to avoid scaring anyone off. I provided additional links to encase more information (there is only so much one post can discuss) should you decide to go further down the rabbit hole.

Hope you enjoy!

## More Equal Than Others
Database queries are not created equal - some are short and simple, some are long, arduous and complicated. Some take an insignificant amount of time and some make our DBA team cry at night. Some are just unlucky and need to wait for some DNS shenanigans, a lazy network card or some mumbo jumbo about [tombstones in SSTables](https://medium.com/walmartglobaltech/tombstones-in-apache-cassandra-d0a068a72dcc). All of them take _some_ amount of time.

While a query is executing, an application worker is waiting.

A waiting worker is a waiting customer.

A waiting customer is an unhappy customer.

We do not want unhappy customers.

A waiting worker is also a waste - computers are generally quite fast these days. Waiting for a _millisecond_ can mean _millions_ of wasted instruction cycles on a modern processor. Query wait times can add up _really, really_ fast. We needed to optimize this wait time, and we needed it yesterday.

## Frankly, my darling, I don't give a _query_
[Cassandra Communicator](https://moo64c.github.io/articles/2021/08/15/Cassandra-A-Scale-y-Story/) already optimize workers' wait time; all **write** queries are performed _without replying to the worker_. Application workers send the `insert` or `update` query to the Communicator, and go on their merry way. In most cases, they do not care if the write query actually succeeded; would they do much more than write something to a log and send a metric if the query failed? Communicator can handle that. Caring about the result equals waiting on a query before doing anything else.

But you always care when reading from the database. Our workers will always wait on **_read_** queries.

**Read** queries produce data, and the worker needs to use the data for something - logic, additional queries or sent elsewhere, maybe just verify that it exists. We can not avoid waiting for the query before doing something with the data before we could carry on.

Or is there?

## In Threads We Trust
Asynchronous (aka async) queries allow for a lot of flexibility when optimizing. One way to enjoy the async performance benefit for read queries is to add a [thread](https://stackoverflow.com/a/5201906) per query to wait on its reply. Threading will work well under the assumption at some point A in the code which requires some data from the database, there is a point B of code that does not require that data at all. Simply: break the flow to tasks. Put each task in some thread's capable hands to fetch and process the data, and the worker itself (main thread) can do other tasks in the meantime - processing data or starting more queries and threads. The limit on concurrent threads is how much the processor can handle - running thousands of them at the same time is theoretically possible. When you run out of code to run without the queried data, the main worker thread will have to block until all query threads finish (aka _join_ back to the main thread).

![Sending queries into their own threads and joining the main thread later.](../assets/images/2022-08-20-Query%20Grouping/threading.png)

Adapting existing code, which means splitting the queries into threads, requires more than bit of work. Most of it is finding queries that fit this use case in the code and where they need to _join_ back to the main thread. Additionally, anything done inside a thread must be [thread-safe](https://en.wikipedia.org/wiki/Thread_safety), since at any given point in the code we can stop and start running some other piece of code with the same state. Every time you add a thread the eternal headache of locking mechanisms is right around the corner.

This makes the optimization come with a hefty cost of complexity in the code. Used incorrectly, the worker can actually end up waiting more - it is up to the developer to find the correct use cases. All in all, threads can be an optimal choice for very specific use cases, such as code that draws a lot of different data sets completely foreign to one another for different uses, but a terrible choice in other use cases.

Threads also add a not-so-hidden cost of [_context switching_](https://en.wikipedia.org/wiki/Context_switch), which might eliminate any potential performance benefit. This is why - as a generic solution - threading did not fit well. More importantly our codebase **does not allow for threading** at all. Everything must happen in the main worker thread or in some other process (like Communicator handling _write_ queries by itself).

Luckily there is more than one solution to optimizing query wait time. Back to the drawing board.

## Come Together, Right Now
Client code for Communicator was implemented with the existing code in mind, meaning every time the worker sends a query to Communicator it blocks until the query completes. Communicator handles itself async queries directly to Cassandra; it made sense to allow the worker to send a **_group_** of queries instead of one. From the worker perspective all the queries would run at the same time, blocking once for all the queries, and taking the group's worst (longest) query's time to execute, which is much better than the sum of time of each individual query. There is a certain overhead cost for grouping up (memory wise), and it also causes sending larger datasets over local TCP sockets - but performance wise it is a very cheap trade off.

![The async mechanism reduces worker wait time by waiting for as many queries as possible at once.](../assets/images/2022-08-20-Query%20Grouping/Sync%20V%20Async.png)

Creating queries in groups sounds like a hassle, like the one described for threading above. What about the engineering cost?

In theory, grouping queries might take prior knowledge of which queries are required in some following logic: queries come first in some prefetch of all data, and the logic comes later. That makes for a very rigid flow of execution, adds complexity and prone to many issues - such as following conditional queries cannot be optimized at all. It is a huge headache to rewrite or refactor our existing code (which was not built with async in mind) to fit this prefetch mentality, and we had no intention of rewriting anything on that scale.

Therefore, the question is: How can we group queries without entering a realm of painful rewrites?

## Don't _thread_ on me
Luckily, threads are not the only answer to doing several tasks in "parallel". A lesser known concept called [Coroutines](https://en.wikipedia.org/wiki/Coroutine) allows single-thread "threading". Hopefully not diving too deeply to the technicalities, the idea is to let functions suspend while other functions do their thing for a bit before returning the first function. Putting names to this process, `Yielding` inside a running coroutine (function) allows other coroutines to `resume`, which in turn `yield` again (or end) and let the other coroutines `resume`. Explicitly - coroutines allow the worker to jump between tasks, resuming any task in the same point it left it.

 It does sound a bit like threading, but it all happens in a single thread. Under the hood threads actually behave in a manner very close to that description, but the [kernel](https://www.redhat.com/en/topics/linux/what-is-the-linux-kernel) handles all the `yielding` and `resuming` (and a lot more). Coroutines expose this functionality directly in user space (application side). [Lua.org](https://www.lua.org) has some [great usage examples](http://www.lua.org/pil/9.4.html) for coroutines for a more technical perspective (and how it is used in Lua), but [this talk](https://www.youtube.com/watch?v=Hi6ICEVVRiw) by Kevlin Henney might be easier to digest than a bunch of Lua code.

Coroutines can be great when waiting for input/output (I/O, i.e. reading from a socket), or when several pieces of code need to reach a similar point before running a costly action (i.e. executing a query). While the socket is waiting for more data in one coroutine, another coroutine can move on and do something else. Since I/O is handled by the kernel, there is no need to babysit it in user space (in the application worker). Coroutines do not require context switching and the kernel is not involved meaning a possible significant performance advantage over threads.

> Discussing slightly more popular languages than Lua, coroutines were added in Python 3.5 (existed before as _generators_). To use coroutines, define a function as `async` - and it will return a `promise` object when called. To get a response, just `await` on the promise object. This allows the runtime to do other things in the meantime, and [when used with the right tool](https://docs.python.org/3/library/asyncio-task.html#coroutine) it can run a very performant networking process; I will discuss a pretty cool use case (in another service) in a future post.

Using coroutines has some other terrific property: any piece of code can become a coroutine. Wrap existing code in a function, and it could run concurrently to other piece of code - just add a `yield` in the proper place. Coroutines never require a [mutex](https://stackoverflow.com/a/34556) on any resources since they do not run in some randomized parallel context like threads - you always know when the coroutine exits and when it resumes. There is no need to rewrite a lot of ancient database models or adjust the flow of our code; only a minimal intervention in existing code is required to make it able to get a bunch of queries into a group and execute it. There is also no need to rewrite a bunch of manual code for every use case; a general use class can be made.

By combining coroutines with grouped queries we can reduce waiting times for our workers with little engineering hassle. Wrapping an existing block of code in a function and letting some mechanism handle when to `resume` or `yield` takes a relatively low engineering cost. We just need to create some interface to handle the coroutines. Each query and its following logic can be kept pretty much the same; we would only need to verify the independence of grouped queries and their following logic from one another, and old code can enjoy new performance benefits.

![Grouped queries with coroutines must wait work to finish before starting the query execution.](../assets/images/2022-08-20-Query%20Grouping/normal_vs_threading_vs_grouped.png)

The diagram above shows grouping with coroutines might be worse than the threading example, but for most use cases (including ours) worker code will run in **_nanoseconds_**, but queries usually take **_milliseconds_**. This means the worker parts (orange in the diagram) are several _orders of magnitude_ of times smaller than any query (red or blue) wait parts. In practice, grouping allows for similar and sometimes even better performance than threading in most cases.
_(Disproving this last statement is left as an exercise for the reader)._

## Making Queries Great Again
A new `query_grouping` interface was built to wrap everything discussed here together. It turns the functions containing the queries to coroutines, `yields` when a query is sent during a `group:run()` call and manages the coroutines to `resume` when their query completes. Instead of sending the query immediately, it groups them together, sending it once all coroutines have `yielded`. With minimal additional engineering effort, `query_grouping` can shave several precious _milliseconds_ from any flow that has many queries. In a common API call it can translate to dozens of milliseconds. Dozens!

In practice, we saw an overall improvement in performance after implementing Query Grouping in specific API calls. It is implemented only for Cassandra through Communicator with plans for expansion, but that might take a while. `query_grouping` also forces developers to name the groups, allowing us to expose configuration to turn a specific query group back to "non-grouped" flow if a problem is detected.

### Fresh Samples - How does Query Grouping look like?
Let's say we have the functions `notify_heroes`,`notify_villans` which have some side effect (`notify(...)`) but do not directly affect each other. Each of the functions contains two queries and does something with their result. These functions run one after the other:

```lua
function notify_heroes()
  local turtles = query_turtles()
  local ninjas = query_ninjas()

  notify(filter_teenage_mutant(ninjas, turtles))
end

function notify_villans()
  local technodrome = query_automobile()
  local shredder = query_main_villan()

  notify(shredder, is_sinking(technodrome))
end

notify_heroes()
notify_villans()
```
In the original code each function would run, query the database twice - making the worker wait on each request - and does something with the results. The fully synchronous case has the most wait time for the worker which would wait **_four_** times.

As the functions and their result do not affect one another, they can all run in a query group. To do so we just need to create a group, add the functions, and run it:
```lua
local group = query_grouping.new_routines_group("notify")

group:add(notify_heroes)
group:add(notify_villans)

group:run()
```
That's it; added callbacks turn automatically into coroutines and `group:run()` runs them, blocking until all coroutines entered a `dead` state (ended). Each function has two queries, four queries will execute using just two groups of queries, Blocking **twice** instead of **four** times, with the time cost of the worst query in each group. If all queries take the same amount of time (they never do, but let us assume), we already reduced the wait time by _**half**_.

> If a function does not query the database, it will never reach a `yield` call, meaning it will run until it is finished. This marks the coroutine as `dead` after one `resume` call.

### The Crème De La Crème:
`notify_heroes` runs two queries which we can assume do not affect one another, and the same could be said for `notify_villans`'s queries. After we grouped `notify_heroes` and `notify_villans`, we block twice for two groups of queries: first group has `query_turtles` and `query_automobile`, and the second group of queries includes `query_ninjas` and `query_main_villan`.

We can do better: create a _nested_ query group inside `notify_heroes` and another one inside `notify_villans`. Yes, you can create **more groups** inside a running group! This allows `query_grouping` to flatten any _nested_ queries to run in one group, meaning waiting once - regardless of nesting depth and amount of queries. The new code would look like this:

```lua
function notify_heroes()
  local group = query_grouping.new_routines_group("heroes")
  local turtles, ninjas

  group:add(function()
   turtles = query_turtles()
  end)

  group:add(function()
   ninjas = query_ninjas()
  end)
  group:run()

  notify(filter_teenage_mutant(ninjas, turtles))
end

function notify_villans()
  local group = query_grouping.new_routines_group("villans")
  local shredder, tehcnodrome

  group:add(function()
   technodrome = query_automobile()
  end)

  group:add(function()
   shredder = query_main_villan()
  end)
  group:run()

  notify(shredder, is_sinking(technodrome))
end

local group = query_grouping.new_routines_group("notify")

group:add(notify_heroes)
group:add(notify_villans)

group:run()
```
The `"notify"` group starts by running `notify_heroes` as a coroutine. It creates the `"heroes"` group and runs it. `"heroes"` group runs the `query_turtles` and the `query_ninjas` coroutines, both adding a query to the grouped request and yielding. The `"heroes"` group then _returns the control back_ to the `"notify"` group, allowing `notify_villans` to do the same with the `"villans"` group and its two coroutines. Once `"villans"` completes, control is once again at the `"notify"` group's hands, and having no more coroutines to run in that cycle, the grouped request is sent, waiting on the reply.

All this means that **executing all four** queries would block only **once**, essentially reducing the wait time to the minimum of the longest query in the group.  If all queries take the same amount of time, we reduced the wait time to **one fourth** of the original.

There is no limit to nesting groups vertically or horizontally - `notify_heroes` could have additional nesting if there were more queries inside it, and `notify_villans` or any other nested function could also nest its own groups and subgroups. You can have a group run right after a different one. You can stash callbacks to run whenever you would like, and nobody could stop you.

> The unit test code for this is absolute bananas - tests are generating entire call trees with multiple layers of coroutines that do X amount of queries and nest groups in all directions.

The nested group support is what makes a big difference in the engineering overhead. You never have to check if you are in a flow that already uses `query_grouping` as the interface can adjust to whatever you throw at it. Using it, you do not have to verify thread safety or handle some weird edge case. Designed carefully, you could even have some results or state from one callback used in a different one, using the queries' `yields` as a guarantee of execution order. The coroutine order of execution in `query_grouping` is completely predictable meaning simplicity in the code.

The only obvious limits to use `query_grouping` is that the callbacks added do not directly depend on one another, and that just depends on the flow of whatever code you are trying to optimize.

## It's Time to Stop
The `query_grouping` implementation suggests that we do not necessarily need to group the queries - at the `group:add(...)` stage, the request can go to Communicator and start doing its thing, shaving additional precious time and actually matching the threaded behavior in the diagram above. It is true - you can send the request and collect it later, but it would require a rewrite of the Communicator client code on our side which seems unlikely considering the potential benefits are slim compared to the benefit we can attain from simply sending everything as a group. The [80-20 rule](https://en.wikipedia.org/wiki/Pareto_principle) applies here; for an additional "20%" performance benefit the engineering cost would be quite high and prone to bugs.

That is it, thanks for reading!

This blog will be posted on my [personal blog](#) and the [Trusteer Engineering](#) blog.

## Credits
Query Grouping was created, designed and supported by Nir Nahum, Adi Meiman and Amir Arbel from Pinpoint R&D Team. Teenage Mutant Ninja Turtles is owned by Nickelodeon.
