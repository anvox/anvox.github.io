---
layout: post
title: Immutable doesn't mean slow
date: 2021-11-02 15:09 +0700
categories: elixir immutable til
---

TIL, Immutable doesn't mean slow

In elixir, to add an item to list, we write like this

```elixir
list1 = [1, 2, 3]
list2 = [0 | list1]
```

If it was me, I have 2 options to implement: clone `list1` or point `list1` to `list1`. Seems, cloning `list1` is prefered, after lots of pain with data changed in ruby 😂 
But if creating `list2` clones `list1`, it'll be slow. Memory consumed, time for GC... Chosing pointing, what if `list1` changed?

Wait!

`elixir` is immutable, that means, `list1` isn't changed, that means, we needn't care about breaking `list2` in this way.

Another point is, elixir list is linked list. So, it's native faster to implement the pointing way. 

----

Extract from [Programming Elixir 1.6](https://pragprog.com/titles/elixir16/programming-elixir-1-6/), Chapter 3
