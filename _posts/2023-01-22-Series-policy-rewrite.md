---
title:  "policy rewrite series"
categories: ["Devlog", "Code Quality"]
tags: ["quality"]
#TODO
excerpt: 
---

Outline for posts:
1. LEAN GUIDE TO CODE REFACTORING (making terrible code less terrible) LUA EDITION
  - Disclaimers
    - Have not invented anything here - I've condensed works of some and cherry-picked from others.
    - Lead up to policy rewrite post.
    - Did not include everything I work by, just what I presume important.
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
  - The explanation behind each rule. (Blab on as much as needed)
  - Appendix: Nobody is perfect.
    - Code review every change
    - Use automated tools to lint, format and check your code (some VS Code extensions for Lua)
      - Lua Linter
      - Lua Server
      - Complexity checker
      - Spellchecker

2. Rewriting a major "haunted forest" feature
  - what is structured policy.
  - Goal and the major blocker.
  - Assumptions (length of change, simplicity, depth)
  - Struggles (line by line refactoring)
    - see the LEAN GUIDE TO CODE REFACTORING
    - Very specific nested samples
  - Being wrong (discoveries - like the underlying rules engine inside policy)
  - Results (bugs, cleaner code)
  - aftermath (what could be done better)

3. Test policy deferring
  - Requirements (deferring a lot of info)
  - Challenges (network traffic, cost, options)
  - Deciding on the local option 
  - Missioner service:
    - stack (ioloop in tornado)
    - Coroutines are awesome, yet again
    - performance and throttling
  - Final results (API performance before and after)