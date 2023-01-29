---
title:  "Un-legacy That Code"
categories: ["Devlog", "Code Quality"]
tags: ["lua", "refactoring", "coding"]
excerpt: "Explaining my approach to refactoring - part one in a series about making our API 10 times faster."
---

This series of post is about an expansion for a feature called 'Policy'. Policy was a piece of code written originally about 8 years ago, absorbing feature after feature over the years, which turned it into a [haunted forest](https://increment.com/software-architecture/exit-the-haunted-forest/). You can not work with haunted forests (per definition), so to get to the next post (part two) which would discuss the rewrite itself, I need to share my methods for tackling refactoring, rewriting, and what I expect when I want to make changes in existing code. I share my main focus in any refactor, the "problems" I seek to fix and the rules for them, and why every rule matters. 

Part three would discuss the expansion which allowed us to improve our response times by a factor up to 10. Additionally, it would include how we saved hundreds of thousands of dollars in AWS money with - you guessed it - a new service. Hopefully this clickbait-y description would help you get through three posts!

## This is Another Public Service Announcement
One of the conclusions I reached in the last post is that I should try to avoid 3000+ words posts - I _could_ go into my professional history here to contextualize a bit of the ideas in this post, but it should suffice to say I have _some_ experience. I learned from - and working with - experienced professionals for over a decade, and devoted more than a fair share of my time to reading books and watching lectures about what makes code good. I know a little about code.

Of course, anything I write in this post is purely subjective. The examples could be useless to anyone who does not use Lua (but they should be generic enough for most programming languages). I feel it is good to share ideas and get feedback (hopefully not telling me I'm _completely_ wrong). It is it very hard to stay focused when discussing (or reading about) refactoring; a solved problem (existing code) is just not as exciting as writing new code. I believe this post could help when writing new code as well as refactoring, and the ideas would hold for both.

With the structure, expectations and disclaimers out of the way, let's start with where I deviate from the "accepted" literature.

## The Annoyance of a Good Example
One of the things I noticed when reading books like [_Clean Code_](https://learning.oreilly.com/library/view/clean-code-a/9780136083238/), [_Five Lines of Code_](https://learning.oreilly.com/library/view/five-lines-of/9781617298318/) and [_Good Code, Bad Code_](https://learning.oreilly.com/library/view/good-code-bad/9781617298936/) is that the author is trying to provide a literal set of rules of what to do or what not to do. I find this kind of mentality difficult. Sometimes rules are good, and sometimes rules tell you to do seriously weird stuff:

![Terrible example of refactoring in Typescript](/assets/images/2023-01-01-Un-legacy-That-Code/do-one-thing.PNG)

<small>Taken from [_Five Lines of Code_ by Christian Clausen, Manning Publications](https://learning.oreilly.com/library/view/five-lines-of/9781617298318/OEBPS/Text/03.htm#:-:text=3).</small>

The idea here is that we make small functions, and the *rule* is that you do one thing, and one thing only in every function. In sub-rule, he explains _where_ `if` blocks should reside; the rule after this one is about avoiding `else` statements completely. Christian is not wrong; a very similar example exists in Clean Code by Robert C. Martin (Uncle Bob), which Christian lists as an inspiration, with a very similar rule in it. I would also agree `else` is terrible in my eyes for code structure; they add a lot of confusion, and when using an "early exit" strategy, they are completely useless.

But it should not be a _rule_. `else` statements have their use, and having a code base with functions that do one _thing_ is a terrible idea. A definition for _thing_ is missing - where do you draw the line? According to this example, one function call or starting an iteration is enough work for a function. With this kind of refactoring you can only do one thing in the entire application - at the end of a long call chain. I cast serious doubts at the _maintainability_ of a codebase written this way. And maintainability is usually the focus of a refactor.

<small>If any of the three books above interest you, I would recommend starting with _Clean Code_. Most examples are in Java, and even though I think there's no such thing as _pretty_ Java code, the ideas are clear and easy to follow.</small>

## More What You'd Call Guidelines Than Actual Rules
I could break the example above further, going on about variable naming and the missing parenthesis in the if statement, but I digress; the idea here is that defining what is good code is purely subjective. I will not be writing any _rules_. Nothing is set in stone and everything is subject to change. Instead, I want to provide a single point to focus and expand on, hoping you would take it with you on your coding adventures from now until you retire and live happily growing vegetables and poultry, never looking at a computer screen ever again.

That point is **_Readability_**.

I use a simple trick with post readability that I keep hearing about whenever I ask for commentary on my posts. Everyone likes the quips, the references like the title above. It helps break down the naturally occurring walls of text in posts like these, and grounds people to certain things. Maybe even helps them remember what I wrote about, who knows. I have been writing


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