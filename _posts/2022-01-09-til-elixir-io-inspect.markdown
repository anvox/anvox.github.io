---
layout: post
title: 'TIL: Elixir IO.inspect'
date: 2022-01-09 16:40 +0700
categories: elixir
---

In my very first journey with Elixir, I use `IO.inspect` most of time. But after a long time, I found it return it returns whole input. It means we could use it pipeline without aware the effect:

```elixir
# Original
do_a
|> do_b
|> do_c

# Debug - Same input to `do_c` 
do_a
|> do_b
|> IO.inspect
|> do_c
```
