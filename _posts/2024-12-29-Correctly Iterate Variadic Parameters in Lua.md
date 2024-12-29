---
title:  "Correctly Iterate Variadic Parameters in Lua"
author: "Amir Arbel"
categories: ["Devlog", "Lua", "how-tos"]
date: "2024-12-29"
tags: [lua]
excerpt: Using `select` to handle a Lua quirk.
---

Our logger functionality exposes a simple interface, which doesn't limit the number of items you can send it. 

```lua
function M.info(...)
function M.error(...)
-- etc.
```

To send it to the syslog service, we need to convert all the items into a single `string`. Normally, you would convert the variadic parameter `(...)` to an array and use `ipairs` to iterate over it:

```lua
local texts = {}
for index, item in ipairs({...}) do
  texts[#texts + 1] = tostring(item)
end
local result = table.concat(texts, " ")
```

However, `ipairs` has a quirk : a `nil` value will cause it to stop the iteration over the array.

```lua
logger.info("foo", nil, "bar")
-- logs "foo"
```

How to fix this? Using `select`!
- `select("#", ...)` retrieves the actual number of items hidden in the variadic parameter.
- `select(i, ...)` will retrieve the item in index `i` (remember they start at 1 in Lua) from the variadic parameter.

For example - 

```lua
local reduce_to_string = function(...)
  local texts = {}
  for index = 1, select('#', ...) do
    texts[#texts + 1] = tostring(select(index, ...))
  end
  return table.concat(texts, " ")
end

```