---
title:  "Query Grouping (title TBD)"
layout: single
categories: ["Devlog", "Lua", "Communicator"]
tags: [backend, database, cassandra, epoll, lua, threading]
---

Outline
  - sync vs async - analogy: bus vs taxi
  - the problem: lots of sync queries in a row
  - co routines: the non-threaded threading
  - query grouping
