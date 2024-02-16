---
title:  "Queries in a Pinch"
categories: ["Devlog", "Lua", "Communicator"]
tags: [backend, database, cassandra, lua, threading, coroutines, async]
excerpt: Smart database reads - speed up an application.
---

This post is about an optimization to response times acquired by changing how application workers wait for database query execution. It discusses why we chose this optimization, what were the available options for a solution, what we chose and how we refactored our code - including some samples. That sounds quite in-depth and technical, and although I kept the code samples to the very end to avoid scaring anyone off, it does presume some prior knowledge. I provided additional links to expose more information for the more curious of readers.
<!--more-->
Hope you enjoy!

## More Equal Than Others
Database queries are not created equal - some are short and simple, some are long, arduous and complicated. Some take an insignificant amount of time and some make our DBA team cry at night. Some are just unlucky and need to wait for some DNS shenanigans, a lazy network card or some mumbo jumbo about [tombstones in SSTables](https://medium.com/walmartglobaltech/tombstones-in-apache-cassandra-d0a068a72dcc).
But all of them take _some_ amount of time. And while a query is executing, an application worker is waiting.

A waiting worker is a waiting customer.

A waiting customer is an unhappy customer.

Nobody wants unhappy customers.

A waiting worker is also a waste - computers are generally [quite fast](https://computers-are-fast.github.io/) these days. Waiting for a _millisecond_ can mean millions of _wasted_ instruction cycles on a modern processor. These wait times add up _really, really_ fast. We need to optimize this, and we need it yesterday.

## Frankly, my dear, I don't give a _query_

Database queries can be generally split into read queries (`select`) and write queries (`insert`, `update` or `delete`). Our client for [Cassandra Communicator](https://moo64c.github.io/articles/2021/08/15/Cassandra-A-Scale-y-Story/) already optimize workers' wait time by defaulting **write** queries to be performed _without replying_ to the worker. An application worker can send the write query to Communicator, and go on their merry way. In most cases, they do not care if the write query actually succeeded; would they do much more than write something to a log and send a metric if the query failed? Communicator can handle that. Needlessly waiting on a query before doing anything else is wasteful.

But **read** queries produce data which the worker uses for something - logic, additional queries or just verify that it exists. We must stop everything and wait on a query and do something with the data before we could carry on.

Must we?

## In Threads We Trust?
One way to enjoy a performance benefit for read queries is to add a [thread](https://stackoverflow.com/a/5201906) per query to wait on its reply. Threading will work well under the assumption at some point A in the code which requires some data from the database, there is a point B of code that does not require that data at all. We can take the code from A to B and run it in a separate thread. When you run out of code to run without the queried data, the main worker thread will have to block until all query threads finish (`join` back to the main thread).

> Simply: break the flow to tasks. Put each task in a new thread's capable hands to fetch and process the data, and the worker itself (main thread) can do other tasks in the meantime, like starting more queries and threads. The limit on concurrent threads is how much the processor can handle.

![Sending queries into their own threads and joining the main thread later.](https://moo64c.github.io/assets/images/2022-08-20-Query%20Grouping/threading.png?raw=true)

Adapting existing code - splitting the queries into threads - requires more than bit of work. Most of it is finding queries that fit this use case in the code and where they need to `join` back to the main thread. Anything done inside a thread must be [thread-safe](https://en.wikipedia.org/wiki/Thread_safety), since at any given point in the code we can stop and start running some other piece of code with the same state; every time you add a thread the dreaded headache of [locking mechanisms](https://stackoverflow.com/a/34556) is right around the corner. Threads also add a not-so-hidden cost of [_context switching_](https://en.wikipedia.org/wiki/Context_switch), which might eliminate any potential performance benefit.

This makes the optimization come with a hefty cost of complexity in the code. Used incorrectly, the worker can actually end up waiting more - it is up to the developer to find the correct use cases. All in all, threads can be an optimal choice for very specific use cases, such as code that draws a lot of different data sets completely foreign to one another for different uses, but a terrible choice in other use cases.

Due to the reasons listed above, we did not consider threading at all for our use case. Our codebase does not allow for threading without jumping through additional hoops and bottlenecks since it is already multithreaded; each application worker is a thread which has a Lua state machine, which dislikes running additional threads. Everything must happen in the worker thread or in some other process (like Communicator handling _write_ queries by itself).

Luckily there is more than one solution to optimizing query wait time. Back to the drawing board.

## Come Together, Right Now
Client code for Communicator was implemented to block when sending a query to Communicator; backwards compatibility required it. Communicator itself runs the queries asynchronously; it can handle multiple queries at any given moment without blocking. It made sense to allow the worker to send a **_group_** of queries instead of just one.

From the worker perspective all the queries would run at the same time, blocking once for all the queries, and the query group taking the group's worst (longest) query's time to execute, which is much better than the sum of time of each individual query. There is a certain overhead cost for grouping up (memory wise), and it also causes sending larger datasets over local TCP sockets - but performance wise it is a very cheap trade off.

![The async mechanism reduces worker wait time by waiting for as many queries as possible at once.](https://moo64c.github.io/assets/images/2022-08-20-Query%20Grouping/Sync%20V%20Async.png?raw=true)

Changing the queries in the code to group up sounds like a hassle, like the one described for threading above. What about the engineering cost?

In theory, grouping queries might take prior knowledge of which queries are required in some following logic: queries come first in some prefetch of all data, and the logic comes later. That makes for a very rigid flow of execution, adds complexity and prone to many issues - such as the following conditional queries cannot be optimized at all. It is a huge headache to rewrite or refactor our existing code to fit this prefetch mentality, and we had no intention of rewriting a lot of existing code down to this level.

How can we group queries without entering a realm of painful rewrites?

## Don't _thread_ on me
Luckily, threads are not the only answer to doing several tasks in "parallel". A lesser known concept called [Coroutines](https://en.wikipedia.org/wiki/Coroutine) allows single-thread "threading". The idea is to let functions suspend while other functions do their thing for a bit before returning the first function. Putting names to this process, `Yielding` inside a running coroutine (function) allows other coroutines to `resume`, which in turn `yield` again (or end) and let the other coroutines `resume`. Explicitly - coroutines allow the worker to jump between tasks, resuming any task in the same point it left it.

It does sound a bit like threading, but it all happens in a single thread. Under the hood threads actually behave in a manner very close to that description, but the [kernel](https://www.redhat.com/en/topics/linux/what-is-the-linux-kernel) handles all the `yielding` and `resuming` (and a lot more). Coroutines expose this functionality directly in user space (application side). [Lua.org](https://www.lua.org) has some [great usage examples](http://www.lua.org/pil/9.4.html) for coroutines for a more technical perspective (and how it is used in Lua), but [this talk](https://www.youtube.com/watch?v=Hi6ICEVVRiw) by Kevlin Henney might be easier to digest than a bunch of Lua code.

> Discussing slightly more popular languages than Lua, coroutines were added in Python 3.5 (existed before as _generators_). To use coroutines, define a function as `async` - and it will return a `promise` object when called. To get a response, just `await` on the promise object. This allows the runtime to do other things in the meantime, and [when used with the right tool](https://docs.python.org/3/library/asyncio-task.html#coroutine) it can run a very performant networking process; I will discuss a pretty cool use case (another service) in a future post.

Coroutines can be great when waiting for input/output (I/O, i.e. networking - reading from a socket), or when several pieces of code need to reach a similar point before running a costly action (i.e. executing a group of queries). Instead of waiting on a socket in one coroutine, `yield`, and another coroutine can move on and do something else. Since I/O is handled by the kernel, there is no need to babysit sockets in user space (in the application worker). Coroutines do not require _context switching_ meaning a significant performance advantage over threads.

Using coroutines has some other terrific property: any piece of code can become a coroutine. Wrap existing code in a function, and it could run concurrently to other piece of code - just add a `yield` in the proper place. Coroutines never require a mutex (lock) on any resources since they do not run in some randomized parallel context like threads; you always know when the coroutine exits and when it resumes. There is no need to rewrite a lot of ancient database models or adjust the flow of our code; only a minimal intervention in existing code is required to make it able to get a bunch of queries into a group and execute it. There is also no need to rewrite a bunch of manual code for every use case; a general use class can be made.

By combining coroutines with grouped queries we can reduce waiting times for our workers with little engineering hassle. Wrapping an existing block of code in a function and letting some mechanism handle when to `resume` or `yield` takes a relatively low engineering cost. We just need to create that mechanism. Each query and its following logic can be kept pretty much the same; we would only need to verify the independence of grouped queries and their following logic from one another, and old code can enjoy new performance benefits.

![Grouped queries with coroutines must wait work to finish before starting the query execution.](https://moo64c.github.io/assets/images/2022-08-20-Query%20Grouping/normal_vs_threading_vs_grouped.png?raw=true)

The diagram above shows grouping with coroutines might be worse than the threading example, but for most use cases (including ours) worker code will run in **_nanoseconds_**, but queries usually take **_milliseconds_**. This means the worker parts (orange in the diagram) are several _orders of magnitude_ of times smaller than any query (red or blue) wait parts. In practice, grouping with coroutines allows for similar and sometimes even better performance than threading.

_Disproving that last statement is left as an exercise for the reader._ Have a go at it in the comments below!

## Making Queries Great Again
A new `query_grouping` interface was built to wrap everything discussed here together. It turns the functions containing the queries to coroutines, `yields` when a query is sent during a `group:run()` call and manages the coroutines to `resume` when their query completes. Instead of sending the query immediately, it groups them together, sending it once all coroutines have `yielded`. With minimal additional engineering effort, `query_grouping` can shave several precious _milliseconds_ from any flow that has many queries. In a common API call it can translate to dozens of milliseconds. Dozens!

In practice, we saw an overall improvement in performance after implementing `query_grouping` in specific API calls. It is implemented only for Cassandra through Communicator with plans for expansion, but that might take a while.

### Fresh Samples - How does Query Grouping look like?
Let's say we have the functions `notify_heroes`,`notify_villains` which have some side effect (`notify(...)`) but do not directly affect each other. Each of the functions contains queries (functions prefixed with `query_`) and does something with their result. These functions run one after the other:

```lua
function notify_heroes()
  local turtles = query_turtles()
  local ninjas = query_ninjas()

  local _, ninja = next(ninjas)
  notify(filter("teenage", "mutant", ninja, turtles))
end

function notify_villains()
  local technodrome = query_automobile()
  local shredder = query_main_villain()

  notify(shredder, is_sinking(technodrome))
end

notify_heroes()
notify_villains()
```
In this code each function would run, query the database twice - making the worker wait on each request - and do something with the results. This somewhat simple case has four queries - each time waiting and immediately sending off another query and waiting again.

![Common flow with multiple independent queries.](https://moo64c.github.io/assets/images/2022-08-20-Query%20Grouping/notify_flow_1.png)

As the functions and their result do not affect one another, they can all run in a query group. To do so we just need to create a group, add the functions, and run it:
```lua
local group = query_grouping.new_routines_group("notify")

group:add(notify_heroes)
group:add(notify_villains)

group:run()
```
Added callbacks turn automatically into coroutines and `group:run()` runs them, blocking until all coroutines entered a `dead` state (function return or error). Since each function has two queries, each would yield twice - and four queries will execute using just two groups of queries. This would block **twice** instead of **four** times, with the time cost of the worst query in each group. If all queries take the same amount of time (they never do, but let us assume), we already reduced the worker wait time by _**half**_.

![Group:run() flow: while coroutines are alive, resume each added coroutine; yield to add queries to a grouped query request. After resuming all once, send the grouped request, repeat.](https://moo64c.github.io/assets/images/2022-08-20-Query%20Grouping/run_flow.png?raw=true)

> Any code that does not query the database will never reach a `yield` call, meaning it will run until the coroutine is `dead`.

![Same flow with a single query group.](https://moo64c.github.io/assets/images/2022-08-20-Query%20Grouping/notify_flow_2.png)

### Crème De La Crème:
`notify_heroes` runs two queries which we can assume do not affect one another, and the same could be said for `notify_villains`'s queries. After we grouped `notify_heroes` and `notify_villains`, we block twice for two groups of queries: first group has `query_turtles` and `query_automobile`, and the second group of queries includes `query_ninjas` and `query_main_villain`.

We can do better: create a _nested_ query group inside `notify_heroes` and another one inside `notify_villains`. Yes, you can create **more groups** inside a running group! This allows `query_grouping` to flatten any _nested_ queries to run in one group, meaning waiting once - regardless of nesting depth or query amount:

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

  local _, ninja = next(ninjas)
  notify(filter("teenage", "mutant", ninja, turtles))
end

function notify_villains()
  local group = query_grouping.new_routines_group("villains")
  local shredder, tehcnodrome

  group:add(function()
   technodrome = query_automobile()
  end)

  group:add(function()
   shredder = query_main_villain()
  end)
  group:run()

  notify(shredder, is_sinking(technodrome))
end
```
The `"notify"` group starts by running `notify_heroes` as a coroutine. It creates the `"heroes"` group and runs it. `"heroes"` group runs the `query_turtles` and the `query_ninjas` coroutines, both adding a query to the grouped request and yielding. The `"heroes"` group then _returns the control back_ to the `"notify"` group, allowing `notify_villains` to do the same with the `"villains"` group and its two coroutines. Once `"villains"` completes, control is once again at the `"notify"` group's hands, and having no more coroutines to run in that cycle, the grouped request is sent, waiting on the reply.

![Same flow with nested query grouping.](https://moo64c.github.io/assets/images/2022-08-20-Query%20Grouping/notify_flow_3.png)

**Executing all four** queries would block only **once**, essentially reducing the wait time to the longest query in the group.  If all queries take the same amount of time, we reduced the wait time to **one fourth** of the original code.

There is no limit to nesting groups vertically or horizontally - `notify_heroes` could have additional nesting if there were more queries inside it, and `notify_villains` or any other nested function could also nest its own groups and subgroups. You can have a group run right after a different one. You can stash callbacks to run whenever you would like, and nobody could stop you.

The nested group support is what makes a big difference in the engineering overhead. You never have to check if you are in a flow that already uses `query_grouping` as the interface can adjust to whatever you throw at it. Using it, you do not have to verify thread safety or handle weird edge cases.

The only obvious limits to use `query_grouping` is that the callbacks added do not directly depend on one another, and that just depends on the flow of whatever code you are trying to optimize.

## It's Time to Stop
The `query_grouping` implementation suggests that we do not necessarily need to group the queries - at the `group:add(...)` stage, the request can go to Communicator and start doing its thing, shaving additional precious time and actually matching the threaded behavior in the diagram above. We could have designed the request to be completely asynchronous, but it would have required a rewrite of the Communicator client code - forcing us to rewrite much of the existing code as well. Considering the potential benefits are somewhat slim compared to the overhead required to accomplish that. The [80-20 rule](https://en.wikipedia.org/wiki/Pareto_principle) applies here; for a potential 20% benefit, 80% of the work is yet to be done.

Like with Communicator - a good place to stop optimizing and to ship the feature was obvious.

### Conclusion

That is it for this (~3000 words) post. We tackled a common optimization issue (waiting on I/O) with an approach that fits an existing code base. We gained performance in multiple flows easily, adding as little complexity as possible, and we made a pretty awesome interface to handle everything. Hope you enjoyed!

Thanks for reading!

This blog will be posted on my [personal blog](https://moo64c.github.io/articles/2022/12/22/Queries-in-a-Pinch/) and the Trusteer Engineering blog.
