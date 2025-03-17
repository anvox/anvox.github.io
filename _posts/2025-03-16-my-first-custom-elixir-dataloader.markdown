---
layout: post
title: My first custom elixir dataloader
date: 2025-03-16 22:19 +0700
categories: elixir dataloader experiment til
---

I could safely assume that, if you use absinthe with ecto, you are almost familiar with dataloader. The idea is simple, defering query data queries, mostly, for relations, then query in batch at once. It integrates smoothly with ecto relations that I sometime thought it's natually magic. 

Until some anti-patterns come. I'm not sure if these anti-patterns are traditional approaches, or are they fit to the context. But trying to make run leads me to some funny experiments and allowed me understand absinthe, dataloader better. Let's move on. 

## One way rellation

Consider schemas. Normally, we have product has_one profile, and vs profile belongs_to product. 
But in my graph, `External` depends on `Products` and `Products` should know nothing about `External`, even the module name. Every business functions will be kept on `External`. 

```elixir
defmodule Products.Product do
  schema "products" do
    # Lots of fields
    # has_one(:external_profile, External.Profile)
  end
end

defmodule External.Profile do
  alias Products.Product

  schema "external_profiles" do
    belongs_to(:product, Product)
  end
end
```

Thing went well, until exposing to graphql, I had 2 options, creating a root query for profile, or making it a product's field - which is obviously, clearer and lesser work. But without `has_one` - there's no magic powder to let absinthe know how to load it. So I had to make my own magic powder. It was the point I unveiled the not-so-magic of dataloader. 

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
        {
          :ok,
          Dataloader.get(loader, __MODULE__, {:one, __MODULE__}, product_id: product_id)
        }
      end)
    end
  end
```

In the end, it's just 2 parts: loading and getting. The most important here is the way we extract the key to load and get. In this case, the product_id from profile. Without `has_one`, we must define it ourself. 

## Array UUID field

The same approach when I once tried to load list of participants from list of participant id. 

```elixir
defmodule Order do
  schema "orders" do
    # ...
    field :participants, {:array, Ecto.UUID}
  end
end
```

Having a link table is overkill this case. We only need to show and filter the order by participants sometimes. 
Before, I returned list of id, then client make another query to get list of participants. Filtering is even easier with array of uuid, no joining, just a nested query. 
To eliminate the 2nd query from client, I applied the custom dataloader. Similar to case above, replace load/get by load_many/get_many in this case, and the way we get the keys - participant_ids. 

```elixir
def dataloader() do
  fn parent, _args, %{context: %{loader: loader}} ->
    participant_ids =
      case parent do
        %{participants: ids} -> ids
        _ -> []
      end

    loader
    |> Dataloader.load_many(:default, Participant, participant_ids)
    |> on_load(fn loader ->
      participants = Dataloader.get_many(loader, :default, Participant, participant_ids)

      {:ok, participants}
    end)
  end
end
```

By this, order.participants is list of participants. The code on frontend is simple now, and I could remove the participants query in the root. 

## Conclusion

To make a conclusion, I believe it's fun to write custom dataloader. But as an anti-pattern, too many custom dataloader will cause you headache. When looking back the first case, I still regret that a single `has_one` relation could save lots of custom code 😂.
