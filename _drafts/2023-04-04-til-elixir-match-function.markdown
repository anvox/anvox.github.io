---
layout: post
title: My first custom elixir dataloader
date: 2022-10-31 12:19 +0700
categories: elixir dataloader
---

```elixir
  def dataloader() do
    fn parent, _args, %{context: %{loader: loader}} ->
      product_id =
        case parent do
          %Product{id: product_id} -> product_id
          %{product_id: product_id} -> product_id
          _ -> nil
        end

      loader
      |> Dataloader.load(__MODULE__, {:one, __MODULE__}, product_id: product_id)
      |> on_load(fn loader ->
        profile =
          loader
          |> Dataloader.get(__MODULE__, {:one, __MODULE__}, product_id: product_id)

        {:ok, profile}
      end)
    end
  end
```
