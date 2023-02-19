---
title:  "Un-legacy That Code: Structured Policy Series, Part I"
categories: ["Devlog", "Code Quality"]
tags: ["lua", "refactoring", "coding"]
excerpt: "Explaining my approach to refactoring - part one in a series about making our API 10 times faster."
---

Originally written about 8 years ago, _Structured Policy_ is a massive and central feature in my company's main product. This post is a first in a series is about an ability I was asked to add to it, which will improve our API response times _massively_. To achieve that, I had to rewrite a major part of _Policy_, which unfortunately grew into a [haunted forest](https://increment.com/software-architecture/exit-the-haunted-forest/). 

In this post I will explain what "rewrite" means; I will share my main focus writing any piece of code and some tricks to improve code quality easily. The next post would discuss the feature itself and what I've learned from rewriting it. Part three would discuss the ability itself. It would also include how the development team saved (on paper) hundreds of thousands of dollars in theoretical networking cost with one simple trick.

## This is Another Public Service Announcement
I feel it is good to share ideas and get feedback (hopefully _not all of them_ telling me I'm completely wrong). "Good code" is subjective; it is easy to detect when a code is terrible, but it is very hard to create a piece of code which will be great in anyone's eyes. This post is about a bunch of tips and tricks I use to increase my code's quality, but why should you listen to me?

The ideas in this post come from more than a decade of experience as a developer in different software fields. I want to share ideas; communication is key here. Communication between two team members working on project together; communication with a non-technical, frustrated customer and a tech lead completely baffled by a bug at 2 AM; communication between a developer and their future selves, which remembers none of the code laid out in front of them. I have learned what works in code in different disciplines of software.


## More What You'd Call Guidelines Than Actual Rules
I do not want to think of what I write here as _rules_. There are places where you would want a function to do more than one thing, even if books like [_Clean Code_](https://learning.oreilly.com/library/view/clean-code-a/9780136083238/) would deem otherwise. It does not make sense to carpet-bomb one a rule, which is great in a specific use case, to the rest of your code base. I want to focus on a single point and expand on it - hoping coders around the world use it in their coding adventures from now until they retire and live happily growing vegetables and poultry, never looking at a computer screen ever again.

And **_Readability_** is that single point of focus.

When working on software, especially long-running projects and products, you _read_ a lot more code than _write_. Understanding what is happening, where an issue occurs or what to change to add a new ability becomes the single must time-consuming thing a software developer does. By reducing some code's reading time we are increasing that code's _quality_.

I use a simple, obvious trick with my posts and whenever I ask for commentary on my posts, I always hear the same thing: people like the quips, the references like the subtitle above. It helps break down the naturally occurring walls of text in posts like these - it helps people relate and enjoy a break in the technical mumbo jumbo. Hopefully even helps them remember what I wrote about, but I might be exaggerating.

The purpose of the trick, as mentioned above, is to increase _readability_ between sections. A sentence from some old pirate movie or some old quote and the slate is clean, we can tack on more information. The [_cognitive load_](https://en.wikipedia.org/wiki/Cognitive_load) is being reset by sending the reader to figure out what I'm referencing, blanking out the minutia of the previous section. In code, cognitive load needs to be constantly managed. Every variable and condition is carried along in your head for a journey down a branching path of code execution. And understanding things becomes hard quickly.

## Ain't Nobody Got Time For That
I assume you did not drop everything and read that Wikipedia article, so I will un-academically paraphrase it here (with my interpretation). Humans have limited mental capacity, and it is categorized to three types. Developers wishing to understand a piece of code will add *Germane* load as they process and build their understanding of the flow of code. *Extraneous* load is added when they need to process the _presentation_ of code (i.e. read the code). By reducing the *Extraneous* cognitive load we take on us when reading any piece of code, we increase our capacity for *Germane* cognitive load, allowing us to hold more of the process in our head at any given point.

In plain English, if reading something is _simple_ we can understand more from it, and _faster_.

That is the theory. The third type of cognitive load is *Intrinsic*, which relates to actually solving problems, and that has more to do with _writing_ code. As the final side note of my un-academic paraphrasing of that article, it is noted that heavy cognitive load tends to lead to errors. From the paragraph above we can two things that help us work faster with code:
- Using simpler language and structure.
- Trying to use little as possible of the reader's _working memory_.

All that was said here so far - simple is good. How do we implement it?

## 

--------------

- Complexity can be measured in code. [Cyclomatic complexity](https://en.wikipedia.org/wiki/Cyclomatic_complexity) is a pretty common measure, and easy to calculate. Having as little complexity as possible is a good first step in the path to readability. Breaking code up to tiny pieces is the defeater of high complexity in any one part.
- Consistency in structure and formatting helps us reduce cognitive load by knowing _what to expect_ when you open a file.

- The Goal: Reducing Cognitive Load
  - Benchmarking cognitive load
    - Cyclomatic complexity
    - What you expect to see in a block of code
- How we get readability: (*Short* paragraph)

Visually Appealing Code
  * Sensible function calls
  * You can always move this to a new function
  * Break up nesting
  * The endless condition statement
  * Consistency

The Real Problem: Naming
  - How we name things?
    - Concise
    - Unique
    - Bounded context
  - Do not repeat yourself
  - Spelling

Generally Good Ideas
  * Everything should be private, unless stated otherwise. 
  * No surprises
  * Commenting
  * Use automated tools to lint, format and check your code (some VS Code extensions for Lua)
    - Linter
    - Complexity checker
    - Spellchecker

Additional Advantages
  - Code review every change
- Case study