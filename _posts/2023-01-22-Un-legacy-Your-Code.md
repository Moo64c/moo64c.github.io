---
title:  "Un-legacy That Code: Structured Policy Series, Part I"
categories: ["Devlog", "Code Quality"]
tags: ["lua", "refactoring", "coding"]
excerpt: "Explaining my approach to refactoring - part one in a series about making our API 10 times faster."
---

Originally written about 8 years ago, _Structured Policy_ is a massive feature, growing over the years and turning into a [haunted forest](https://increment.com/software-architecture/exit-the-haunted-forest/). This series is about an additional ability I needed to tack onto it; one which will improve our API response times _massively_. To achieve that, I had to rewrite a major part of _Policy._ In the next post (part two) which would discuss the rewrite itself. This post will be about my methods for tackling this rewrite: refactoring, mentality and what I expect when I want to make changes in existing code. I will share my main focus when I write any piece of code and some common issues in code quality I tend to fix.

Part three would discuss the expansion which allowed us to improve our response times by a factor up to 10. It would also include how the development team saved (on paper) hundreds of thousands of dollars in AWS money with - you guessed it - one simple trick. Hopefully this description would help you get through three posts!

## This is Another Public Service Announcement
One of the conclusions I reached in the last post is that I should try to avoid 3000+ words posts - I _could_ go into my professional history here to contextualize a bit of the ideas in this post, but it should suffice to say I have _more than some_ experience. I have learned from and have been working with experienced professionals for over a decade. I also devote more than a fair share of my time to reading books and watching lectures about what makes code good. With that said, anything I write in this post is obviously purely subjective. The examples could be meaningless to anyone who does not use Lua (but they should be generic enough for most programming languages). I feel it is good to share ideas and get feedback (hopefully not all telling me I'm _completely_ wrong). 

Personally I find it very hard to stay focused when discussing (or reading about) refactoring; a solved problem (existing code) is just not as exciting as writing new code. My intention for this post is to help when writing new code as well as refactoring, and the ideas would hold for both.

With the structure, expectations and disclaimers out of the way, let's start with where I deviate from the "accepted" literature.

## The Annoyance of a Good Example
One of the things I noticed when reading books like [_Clean Code_](https://learning.oreilly.com/library/view/clean-code-a/9780136083238/), [_Five Lines of Code_](https://learning.oreilly.com/library/view/five-lines-of/9781617298318/) and [_Good Code, Bad Code_](https://learning.oreilly.com/library/view/good-code-bad/9781617298936/) is that the author is trying to provide a literal set of rules of what to do or what not to do. I find this kind of mentality difficult. Sometimes rules are good, and sometimes rules tell you to do seriously weird stuff:

![Terrible example of refactoring in Typescript](/assets/images/2023-01-01-Un-legacy-That-Code/do-one-thing.PNG)

<small>Taken from [_Five Lines of Code_ by Christian Clausen, Manning Publications](https://learning.oreilly.com/library/view/five-lines-of/9781617298318/OEBPS/Text/03.htm#:-:text=3).</small>

The idea here is that we make small functions, and the *rule* is that you do one thing, and one thing only in every function. In sub-rule, he explains _where_ `if` blocks should reside; the rule after this one is about avoiding `else` completely. Christian is not wrong; a very similar example exists in Clean Code by Robert C. Martin (Uncle Bob), with a similar rule in it. I would also agree `else` is terrible; they add a lot of confusion, and when using an "early exit" strategy (discussed later), they are completely useless.

But it should not be a _rule_. `else` has its uses. A code base with functions that do one _thing_ is a terrible idea. The definition for _thing_ is absent in both books - where do you draw the line? According to this example, one function call or starting an iteration is enough work for a function. With this kind of refactoring you can only do one thing in the entire application - at the end of a long call chain. I cast serious doubts at the _maintainability_ of a codebase written this way.

<small>If any of the three books above interest you, I would recommend starting with _Clean Code_. Most examples are in Java, and even though I think there's no such thing as _pretty_ Java code, the ideas are clear and easy to follow. I think after that [_The Pragmatic Programmer_](https://learning.oreilly.com/library/view/the-pragmatic-programmer/9780135956977/) is the better choice for a follow-up than the other two.</small>

## More What You'd Call Guidelines Than Actual Rules
The example above can be broken down further, but I would digress. I do not think _rules_ are the way to good code; there are places where you would want a function to do more than one thing. It does not make sense to carpet-bomb one objectively good use case to the rest of your code base. Instead, I want to focus on a single point and expand on it, hoping you would take it with you on your coding adventures from now until you retire and live happily growing vegetables and poultry, never looking at a computer screen ever again.

And **_Readability_** is that single point of focus.

I use a simple, obvious trick with my posts and whenever I ask for commentary on my posts, I always hear the same thing: people like the quips, the references like the subtitle above. It helps break down the naturally occurring walls of text in posts like these - it helps people relate and enjoy a break in the technical mumbo jumbo. Hopefully even helps them remember what I wrote about, but I might be exaggerating.

The purpose of the trick, as mentioned above, is to increase _readability_ between sections. A sentence from some old pirate movie or some old quote and the slate is clean, we can tack on more information. The [_cognitive load_](https://en.wikipedia.org/wiki/Cognitive_load) is being reset by sending the reader to figure out what I'm referencing, blanking out the minutia of the previous section. In code cognitive load needs to be constantly managed. Every variable and condition is carried along in your head for a journey down a branching path of code execution. And understanding things becomes hard quickly.

## Ain't Nobody Got Time For That
I assume you did not drop everything and read that Wikipedia article, so I will un-academically paraphrase it here (with my interpretation). Humans have limited mental capacity, and it is categorized to three types. Developers wishing to understand a piece of code will add *Germane* load as they process and build their understanding of the flow of code. *Extraneous* load is added when they need to process the _presentation_ of code (i.e. read the code). By reducing the *Extraneous* cognitive load we take on us when reading any piece of code, we increase our capacity for *Germane* cognitive load, allowing us to hold more of the process in our head at any given point.

In plain English, if reading is _simple_ we can understand more things _faster_.

That is the theory. The third type of cognitive load is *Intrinsic*, which relates to actually solving problems, and that has more to do with _writing_ code. As the final side note of my un-academic paraphrasing of that article, it is noted that heavy cognitive load tends to lead to errors. From the paragraph above we can extract two major points for faster understanding of code:
- Simpler structure and language.
- Little to zero usage of the human _working memory_.

Enough theory, how do can we make code readable?

## 

--------------

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