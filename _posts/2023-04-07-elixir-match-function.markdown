---
layout: post
title: Postgresql counting over
date: 2023-04-07 09:55 +0700
categories: elixir til match? pattern-matching
---

## Context

When processing in batch, I get a lot cases just remove the invalid item in list. Like this.

```elixir
estimations
|> Enum.filter(fn
  %{"total_price" => _} -> true
  _ -> false
end)
```

In the beginning, it was just fine. But after I wrote the 10th block like this, it annoyed me. Most of case, the filter function feed to `Enum.filter/1` always the same, the only different part is a pattern, why don't we just define a function `filter_by_pattern(pattern)`? Just return `true` or `false` :3 if matched.
It's turn out I was late, I'm not the one think about it. Other languages support pattern matching use a `match` function, why don't `elixir` - a modern-pure-pattern-mattching-based language support `match` function?

[https://hexdocs.pm/elixir/1.14/Kernel.html#match?/2](https://hexdocs.pm/elixir/1.14/Kernel.html#match?/2)

Supported from early versions, so we could write shorter.

```elixir
estimations
|> Enum.filter(&match?(%{"total_price" => _}, &1))
```

## tl;dr

* Elixir suppports `match?/2` to check match or not, which fit exactly to `Enum.filer/1`
* Even though, it's not 100% the same. [https://hexdocs.pm/elixir/1.14/Kernel.html#match?/2](https://hexdocs.pm/elixir/1.14/Kernel.html#match?/2) points out the differences already.
* Less code, less bugs. When you get bored of repeatedly code, find away to avoid it, most of case, someone else did it.

