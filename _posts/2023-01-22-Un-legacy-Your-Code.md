---
title:  "How to Unlegacy Your Code _(Lua Edition)_ or: Detect Fraud Faster Please, Part I"
categories: ["Devlog", "Code Quality"]
tags: ["lua", "refactoring", "coding"]
excerpt: How to rewrite a piece of code nobody wants to touch anymore.
---

This post is the first of three about an expansion to a major feature called `structured policy` (or just `policy`). To accomplish any change to this feature, which was in a [haunted forest](https://increment.com/software-architecture/exit-the-haunted-forest/) state, it went through a long rewrite. In this post I'll share my method for tackling this legacy code - how I modify existing code to provide what I consider good readability. I share my main focus in any code rewrite, the "problems" I seek to fix and the "rules" for them, and why every rule matters. The first part ends with a case study about a more recent rewrite to a different feature called `session_timers` (spoiler alert: it becomes `request_timer`).

Part two would discuss the `policy` feature and its rewrite.
Part three would discuss the expansion. What was the goal and how we saved hundreds of thousands of dollars in AWS money with - you guessed it - a new service.

## The Annoyance of a Good Example
At the beginning of my professional career - A bit after the first iPad was released - I started working for a ~~ tiny company that builds websites ~~ [_boutique web application shop_](https://www.gizra.com/). I did not know it at the time, but the [CTO](https://github.com/amitaibu) is nothing short of a world-renowned _Drupal_ "guru" (the Drupal community would invite to have him present in their conferences). Just starting out in the software world, I stumbled head first into one of the best mentors a junior developer could ask for.

Since we were a five-person company at the time, he would be the one to check my _merge requests_, rightfully decimating them for bad function calls, spelling errors, awkward structuring and terrible naming - among other things. In between tasks, I would get time to explore and contribute (on company time) to open source projects, which have their own set of meticulous maintainers and their demands. Being exposed to those methods and structure of work, the meticulousness and adherence to details and procedures was one of the quickest personal professional jumps I could have achieved in such short time span. Quality is always said to be important, but delivering _something_ is always more important. At Gizra, quality trumped, even in the _"did it because we were young and needed money"_ projects.

Fast-forward a decade and few jobs later, I get to work on major expansion for a major feature, which turns into a rewrite and expansion once I take a peek at the code and run back screaming to my managers. Code that was built over ten years by a dozen (or so) developers, all of them now in other companies. Code that was barely documented in over ten years.

### *_Legacy code._*


## Here be Dragons
A few disclaimers: 
  - The following rules and methods are the ones I try to use daily; everything is subjective. I heard people arguing about tabs versus spaces for years and years. The true rule is consistency (of structure and formatting) - it trumps every claim.
  - The following ideas are not new. Where possible, I'd link/reference where I got the idea from.
  - Not everything I work by is included here; as always, the post is quite long, and it's impossible to gather everything to one _readable_ document.

## *_The Code_* is More What You'd Call Guideline, Than Actual Rules
In recent years I turned to examining the process others present in books, videos and lectures to inch slowly towards the thought process _Gizra_ has. There is a lot of code experts out there; the Uncle Bobs, the gang of four, the [Rockstars](https://dylanbeattie.net/) and [the singleton haters](http://kevlin.tel/). When it comes to refactoring they say the same thing in different flavors (even if some of them use terrible class structuring to showcase it). [Good Code, Bad Code](https://learning.oreilly.com/library/view/good-code-bad/9781617298936/OEBPS/Text/Ch-01.htm#:-:text=The%20six%20pillars%20of,and%20test%20it%20properly.) by Tom Long (Manning Publications) summarizes the general idea pretty well:

> ### The six pillars of code quality:
>   1. Make code readable.
>   2. Avoid surprises.  
>   3. Make code hard to misuse.
>   4. Make code modular.
>   5. Make code reusable and generalizable.
>   6. Make code testable and test it properly.

I'd argue that only the first point should be a "pillar", and the other points stem from it:

### Make code readable.

None of these actually touch on the most important point: _Readable code takes the load off the developer._ Our goal in rewriting anything, or writing well, is to allow a developer who has never seen the code to understand it as quickly as possible. Reducing complexity to improve readability in any one piece code translates directly to all of these points; inherently, they try to make you need to remember less when going over the code line by line - breaking things down to very simple "sentences".

Here's the same list with the readability "pillar" _implemented_:
1. Readable code is readable.
2. Readable code would not surprise you.
3. It would be hard to misuse. Readable code fights misuse through simplicity.
4. You'd have to make readable code modular, or you'd end up with a lot of code in one place, which is not readable.
5. Reusable and generalizable is the same as modular.
6. Modular code is testable, or you are doing modules really, really wrong.

In other words, readability reduces our [_cognitive load_](https://en.wikipedia.org/wiki/Cognitive_load) when going over code. If I have to stop reading and explain to myself what a given piece of code does, maybe by pointing at one variable while advancing through the code it is not readable. The cognitive load was too much, my working memory has overflown. Pulling things out of context directly from Wikipedia:

> A heavy cognitive load typically creates error or some kind of interference in the task at hand.

### Therefore, we strive to reduce the cognitive load.

Through experience, the casual book and trial and error I've learned:
- Complexity can be measured in code. [Cyclomatic complexity](https://en.wikipedia.org/wiki/Cyclomatic_complexity) is a pretty common measure, and easy to calculate. Having as little complexity as possible is a good first step in the path to readability. Breaking code up to tiny pieces is the defeater of high complexity in any one part.
- Humans developed language and the brain functionality to support it, which allows us to _assume_ that code that can be read as human language would be more readable. Readable code is just readable sentences.
- Consistency in structure and formatting helps us reduce cognitive load by knowing _what to expect_ when you open a file.




- The Goal: Reducing Cognitive Load
  - Benchmarking cognitive load
    - Cyclomatic complexity
    - What you expect to see in a block of code
  - The tool: Readability
  - How far you should go
- How we get readability: (*Short* paragraph)
  1. Everything should be private, unless stated otherwise. 
  2. You can always move this to a new function
  3. Break up nesting
  4. No surprises
  5. The endless condition statement
  6. Sensible function calls
  7. Comments
  8. Naming, renaming bonanza
    - How we name things?
      - Concise
      - Unique
      - Bounded context
    - Do not repeat yourself
    - Spelling
- Nobody is perfect:
  - Code review every change
  - Use automated tools to lint, format and check your code (some VS Code extensions for Lua)
    - Lua Linter
    - Lua Server
    - Complexity checker
    - Spellchecker
- The explanation behind each rule. (Blab on as much as needed)
  - (the longest part)
- Case study