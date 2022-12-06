---
title:  "title tbd"
layout: single
categories: ["Devlog", "Lua", "Communicator"]
tags: [backend, database, cassandra, lua, threading, coroutines, async]
---

#  Grouping Queries

## ...But Some are More Equal Than Others
Database queries are not created equal. Some are short and simple, some are long, arduous and complicated. Some take a tiny amount of time, some make our DBA team cry at night. Some are just unlucky and need to wait for some DNS shenanigans or a lazy network card or some [mumbo-jumbo about tombstones in SSTables](https://medium.com/walmartglobaltech/tombstones-in-apache-cassandra-d0a068a72dcc). All of them take _some_ amount of time. While the query is executing, a uWSGI worker is waiting. A waiting worker is a waste, and wait time adds up _really, really_ fast. We need to optimize this wait time, and we need it yesterday.

## Frankly, my darling, I don't give a _query_
One way we made [Cassandra Communicator](https://moo64c.github.io/articles/2021/08/15/Cassandra-A-Scale-y-Story/) optimize queries is to perform all **write** queries _without replying to the worker_. A uWSGI worker would just send the `insert` or `update` query to the database, and go on its merry way. They could not care less if the write actually succeeded - and for good reason: would they do much more than **write** something to the log if the write failed? Communicator can handle that. So most **Write** queries no longer bother our workers. Some do care - waiting on a write before doing something else, and that is allowed, just needs to be specified in the query so Communicator replies.

But you always wait for **read** queries.

**Read** queries produce rows of data, and the worker needs to use the data for something - possibly logic, additional queries, or to send it elsewhere. Maybe just verify that it exists. There is no avoiding the fact that the **worker needs to wait** for the query to return and do something with it before it can move on.

Or is there?

## When in Doubt, Add a Thread
Asynchronous (async) queries allow for a lot of flexibility when optimizing. One way to enjoy the async performance benefit for read queries is to add a thread per query to wait on its reply. Threading will work well under the assumption at a given point in the code that requires some data from the database, there is a following piece of code that does not require that data at all. Simply: put off some task while fetching its data, and we can do some other task in the meantime.

This means that the part that needs the queried data can be moved to a thread that does all that waiting on the query (and the task associated with that data) and lets the worker (or main thread) do some other work in the meantime. When there is no more work to be done without the queried data, the worker will block (the thread will _join_ back) until the thread running the query will finish its work.

![Sending queries into their own threads and joining the main thread later.](https://github.com/Moo64c/moo64c.github.io/blob/query_grouping/assets/images/2022-08-20-Query%20Grouping/threading.png?raw=true)

Splitting the queries into threads requires a bit of work - most of it is finding queries that fit this use case in the code and where they need to _join_ back to the main thread. The performance boost comes a hefty cost of complexity. The worker might still wait for some queries - it is up to the developer to find the correct use cases. This makes threads are an optimal choice for very specific use case - code that draws a lot of different data sets completely foreign to one another for different uses.

Not only does the threading solution has a good chance of not optimizing the worker run time, threads add a not-so-hidden cost of [_context switching_](https://en.wikipedia.org/wiki/Context_switch), which might eliminate any potential performance benefit; in addition, every time you add a thread the eternal headache of locking mechanisms is right around the corner. This is why - as a generic solution - threading does not fit well. More importantly working with Lua - especially in the uWSGI context - **does not allow for threading** at all. Everything must happen in the same thread or in some other process (well, like Communicator).

Luckily, there is more than one solution to optimizing query wait time. Lets go back to the drawing board.

## Come Together, Right Now
Synchronous queries will always take at least the sum of all queries' time. Communicator allows us to run queries in parallel - send a group, and they will all run at the same time. Such grouped queries would only takes the time for the worst query in the group to execute and return. There is a certain overhead cost for grouping up and sending larger datasets over network - but it is negligible.

![The async mechanism reduces worker wait time by waiting for as many queries as possible at once.](https://github.com/Moo64c/moo64c.github.io/blob/query_grouping/assets/images/2022-08-20-Query%20Grouping/Sync%20V%20Async.png?raw=true)

Creating queries in groups sounds like a hassle, like the one described for threading above. What about engineering cost?

Grouping queries might take prior knowledge of which queries are required in some following logic, and as such - queries come first in some prefetch, and the logic comes later. Grouping queries in such a way makes for a rigid flow of execution, adds complexity, and is prone to many issues - such as conditional follow up queries cannot be optimized at all. Moreover, it is a huge headache to rewrite or refactor our existing code (which was not built with async in mind) to fit the prefetch mentality.

The question is: Can we group queries without entering a realm of painful rewrites?

## Don't _thread_ on me
Luckily, threads are not the only solution to doing several things in "parallel". A less known concept called [Coroutines](https://en.wikipedia.org/wiki/Coroutine) allows single-thread "threading". Without diving too deeply to the technical details, the idea is to let functions suspend while other functions do their thing for a bit before resuming the original function. `Yielding` inside a running coroutine allows other coroutines to `resume`, which in turn `yield` again (or end) and let the other coroutines `resume`. Explicitly - coroutines allow the worker to jump between tasks, resuming any task in the same point it left it.

> Coroutines were actually a big feature added to Python 3.5 (before that Python had _generators_, which is slight more complex). To use coroutines simply define a function as `async` - and it will return a `promise` object when called. To get a response, just `await` on this promise. This allows the runtime to do other things in the meantime, and [when used with the right tool](https://docs.python.org/3/library/asyncio-task.html#coroutine) it can run a lot of networking stuff simultaneously. But more on that in a future post about another service.

[Lua.org](https://www.lua.org) has some [great usage examples](http://www.lua.org/pil/9.4.html) for a more technical perspective (and how it is used in Lua). However [this talk](https://www.youtube.com/watch?v=Hi6ICEVVRiw) by Kevlin Henney might be easier to process than a bunch of Lua or even Python code.

Coroutines can be great when waiting for I/O (i.e. reading from a socket), or when several pieces of code need to reach a similar point before running a costly action (say, executing a query). While the socket is waiting for more data, the thread can move on and do something else. Since I/O (such as socket read or write) is handled by the kernel, there is no need to babysit it in userspace (in the application worker). Most importantly - coroutines do not require context switching since the kernel is not involved so they have a significant performance advantage over threads.

Using coroutines has some other terrific property: any piece of code can become a coroutine - wrap it up in a function and it could run concurrently to other piece of code, just add a `yield`. Coroutines never require a mutex (lock) on any resources since they do not run in some parallel context like threads. No need to rewrite a whole lot of ancient database models or adjust the flow of our code; there needs to be minimal intervention in existing code to make it able to get a bunch of queries into a group and execute it. There is also no need to rewrite a bunch of code for every use case; a general use class can be made.

Combining coroutines with grouped queries can gain much of the performance lost to waiting on queries. As the previous paragraph hinted, there is not much hassle in wrapping an existing block of code in a function and letting some mechanism handle when to `resume` or `yield`, making the engineering cost quite centralized. Each query is kept as the existing flow of code does not change in regards to certain data points.

![Grouped queries with coroutines must wait work to finish before starting the query execution.](https://github.com/Moo64c/moo64c.github.io/blob/query_grouping/assets/images/2022-08-20-Query%20Grouping/normal_vs_threading_vs_grouped.png?raw=true)

According to the diagram above grouping with coroutines might be worse than the threading example, but in most use cases (including ours) most worker code will run in _nanoseconds_ where any query will take _microseconds_. In practice, grouping allows for similar or better performance than threading in most cases. Disproving this last statement is left as an exercise for the reader.

<!-- todo: better subtitle -->
## How it all fits together
Naturally named, Query Grouping is an interface that wraps everything together. We can turn the functions containing the queries to coroutines, and we can manage the coroutines to resume when their query completes. The only thing left is to force queries to `yield` if they are trying to query in a Query Grouping context, and voila! We can shave several milliseconds from any request involving the database, which is all of them.

Before I jump into the code I will mention that we actually saw an overall improvement in performance after implementing this. We only implemented for Cassandra, and in there are plans to implement query grouping for Mysql, where the benefits would be much much more noticeable in every request. The implementation also forces names on the groups - exposing configuration to disable the grouping using a feature flag if a problem is detected.

### How does it look like?
Lets say we have the functions `f1,f2,f3` (which have no side effects) each of which contain a part that queries the database once and does something with the result. Lets say they all run one after another:
```
  f1()
  f2()
  f3()
```
In the original code each function would run, query the database - making the worker wait on the request - and compute something with the result. The fully synchronous case is the worst case performance wise, and we can improve it.

Since they have no side effects and `f2` does not need any parameters from `f1`, and `f3` does not need any parameters from `f2` or `f1`, they can all run in one group. To do so we just need to create a query group, add the functions, and run the it:
```
  local group = query_grouping_manager.new_routines_group("group1")
  group:add(f1)
  group:add(f2)
  group:add(f3)
  group:run()
```
That is it. Assuming each function has one query, all three will execute in one query group before resuming the execution.

#### But what if `f1` has two queries (`f1_query1`, `f1_query2`) and `f2,f3` have only one each?
```
function f1()
  local result1 = f1_query1()
  local result2 = f1_query2()

  do_something(result1, result2)
end
```
In that case, the first group would complete the queries for `f2` and `f3` after the first run, but `f1` will create a new execution, and `f2` and `f3` will resuming and reach their end (since there are no more queries, they will not `yield`). Assuming the queries in `f1` are not co-dependent (query2 doesn't need anything form query1), we can do better: create a nested query group inside of `f1`:

```
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
In this case, all 4 queries would execute in a single group run, essentially reducing the wait time to a minimum.

#### What if `f3` doesn't even query the database?
If a function does not query the database, it will never reach the _yield_, so it will run until it is finished, and execution will continue after the other functions will be done.


## One more thing
The Query Grouping implementation suggests that we do not necessarily need to group the queries - at the `group:add()` stage, the request can go to Communicator and start doing its thing, shaving additional precious time and actually matching the performance diagram for threads (and the `run` can be a literal `await` call). It is true but it would require a rewrite of the Communicator client code on our side, which is quite unlikely at this stage since its potential benefits are slim compared to the benefit we can attain from simply grouping. The [80-20 rule](https://en.wikipedia.org/wiki/Pareto_principle) applies here; for an additional 10-20% performance benefit the engineering cost would be quite high (and prone to bugs).
