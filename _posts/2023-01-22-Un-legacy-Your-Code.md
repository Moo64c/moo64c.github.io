---
title:  "Un-legacy That Code: Policy Series, Part I"
categories: ["Devlog", "Code Quality"]
tags: ["lua", "refactoring", "coding"]
excerpt: "Explaining my approach to refactoring - part one in a series about making our API 10 times faster."
---

Created almost a decade ago, _Policy_ is a central feature in Pinpoint. A project to improve API response times was handed to me. To complete it, I had to refactor a major part of _Policy_, which unfortunately grew into a [haunted forest](https://increment.com/software-architecture/exit-the-haunted-forest/) over the years. There's a lot to write about this project; I will try to get it all in three posts.

In this post I will explain how I create and refactor code, to sample things I actually did with over 1200 lines of legacy code. I will share my main focus when I write any piece of code and some tricks to improve code quality easily. The second post would discuss the feature itself and lessons learned from this refactor. The third part would discuss the ability itself and how the development team saved (on paper) hundreds of thousands of dollars in theoretical networking cost with one simple trick.

## This is Another Public Service Announcement
I feel it is good to share ideas and get feedback (hopefully _not all of them_ telling me I'm completely wrong). "Good code" as a definition is **subjective**; it is very hard to create a piece of code which will be great in anyone's eyes. Code is a type of communication, not only between human and computer but between human to human, and I argue that the human to human communication is far more important. This post is about a bunch of tips and tricks I use to increase code's ability to communicate with humans.

The ideas (in this entire blog, not just this post) were gathered from more than a decade of experience as a developer in different software fields. Software is communication. Communication between two team members working on project together; communication with a frustrated project manager and a tech lead completely baffled by a bug at 2 AM; communication between a developer and their future selves, which remembers none of the code laid out in front of them. I have learned to communicate with code.

How do I do it?

## More What You'd Call Guidelines Than Actual Rules
I do not want to think of what I write here as _rules_. There are places where you would want a function to do more than one thing, even if books like [_Clean Code_](https://learning.oreilly.com/library/view/clean-code-a/9780136083238/) would deem otherwise. It does not make sense to carpet-bomb one a rule, which is great in a specific use case, to the rest of your code base. I want to focus on a single point and expand on it - hoping coders around the world use it in their coding adventures from now until they retire and live happily growing vegetables and poultry, never looking at a computer screen ever again.

And **_Readability_** is that single point of focus.

When working on software, especially long-running projects and products, you _read_ a lot more code than _write_. Understanding what is happening, where an issue occurs or what to change to add a new ability becomes the single must time-consuming thing a software developer does. By reducing some code's reading time we are increasing that code's _quality_.

I use a simple, obvious trick with my posts and whenever I ask for commentary on my posts, I always hear the same thing: people like the quips, the references like the subtitle above. It helps break down the naturally occurring walls of text in posts like these. It helps people relate and enjoy a break in the technical mumbo jumbo. Hopefully even helps the reader remember what I wrote about.

The purpose of the trick is to increase _readability_ between sections. A sentence from some old movie or some quote and the slate is clean, we can tack on more information. The [_cognitive load_](https://en.wikipedia.org/wiki/Cognitive_load) is being reset by sending the reader to figure out what I'm referencing, blanking out the minutia of the previous section. In code, cognitive load needs to be constantly managed. Every variable and condition is carried along in your head for a journey down a branching path of code execution. And understanding things becomes hard quickly.

## Ain't Nobody Got Time For That
Assuming you did not pause above to read the Wikipedia article, I allow myself some un-academically interpretation of it: Humans have limited mental capacity. There are three types of cognitive load: understanding a piece of code will add *Germane* load to understand a process and flow of code. *Extraneous* load is added by parsing a _presentation_ of some piece of code (processing the code's structure). All types of cognitive load share the same **capacity** - by reducing the *Extraneous* cognitive load we can increase our capacity for *Germane* cognitive load, which means holding more of the process in our head at any given point.

In plain English, if reading something is _simple_ we can understand more from it, and _faster_.

That is the theory. The third type of cognitive load is *Intrinsic*, which relates to actually solving problems, and that has more to do with _writing_ code. As the final side note, heavy cognitive load tends to lead to errors. We can thus deduce (un-academically), that we can read code faster by:

1. Reducing we need to remember about the code.
2. Using a visual structure that is easy to parse.
3. Using simpler language.

Finally, after we defined we want to achieve readability by using these three ideas, we can start discussing _how to do it_.

## The Reader's Paradox
1. How do we reduce we need to remember about the code?

- Reduce commenting. They are a necessary evil; I try to avoid it where possible - comments become stale fast, and stale comments lie about code. They usually also reduce readability even when not stale, forcing us to _remember_ something that is not explained in the code itself.

- One function should express a _task,_ and it should be broken down into subtasks (in functions) if it is _sensible_. We should always prefer concise and self-explanatory functions. Short functions require us to _remember_ very little. A function should make sense as a written paragraph (not more), hopefully a short one. Other writing advice works here, really, like "avoid run-on sentences".

- Yes, break up your functions, even at the cost of performance. You can spare an additional CPU cycle for your team's sanity. Allow yourself to have general purpose, tested and concise functionality available in static libraries, shared across a project. Make your conditions simple, or turn them into separate functions. Functions allow us to _name_ some part of code - explain what it does without a single comment. Here is a condition I had the pleasure of finding in a codebase:

<!--  TODO (find code) -->
```lua
```

What is actually happening here? It is just a comparator for sorting. We want to know if the previous MFA result was good enough to reuse. Anybody, including the writer, who would read this would have to sit down and think what is happening here.


## Hidden in Plain Sight 




--------------

- Complexity can be measured in code. [Cyclomatic complexity](https://en.wikipedia.org/wiki/Cyclomatic_complexity) is a pretty common measure, and easy to calculate. Having as little complexity as possible is a good first step in the path to readability. Breaking code up to tiny pieces is the defeater of high complexity in any one part.
- Consistency in structure and formatting helps us reduce cognitive load by knowing _what to expect_ when you open a file.

- [X] The Goal: Reducing Cognitive Load
  - [ ] Cyclomatic complexity
  - [ ] What you expect to see in a block of code
- [ ] How we get readability: (*Short* paragraph)

Visually Appealing Code
  * Sensible function calls
  * You can always move this to a new function
  * Break up nesting
  * The endless condition statement
  * Consistency

The Real Problem: Naming
  - [ ] How we name things?
    - [ ] Concise
    - [ ] Unique
    - [ ] Bounded context
  - [ ] Do not repeat yourself
  - [ ] Spelling

Generally Good Ideas
  * Everything should be private, unless stated otherwise. 
  * No surprises
  * Commenting
  * Use automated tools to lint, format and check your code (some VS Code extensions for Lua)
    - [ ] Linter
    - [ ] Complexity checker
    - [ ] Spellchecker

Additional Advantages
  - [ ] Code review every change
- [ ] Case study



There's a whole load of books dedicate to this subject. This is just a (hopefully) sub-3000 words post. There's only so much that is possible to shove in this somewhat digestible format. I implore anyone in the software world to read more about code quality than the occasional post here and there. It will make your work easier.
