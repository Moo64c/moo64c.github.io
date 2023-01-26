---
title:  "Un-legacy Your Code (Lua Edition)"
categories: ["Devlog", "Code Quality"]
tags: ["quality", "refactoring", "legacy"]
excerpt: A journy in rewriting that piece of code nobody wants to rewrite.
---



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