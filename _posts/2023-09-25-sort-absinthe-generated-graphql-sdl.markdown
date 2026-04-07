---
layout: post
title: Sort Absinthe Generated GraphQL SDL
date: 2023-09-25 00:55 +0700
categories: absinthe graphql-sdl
---

## The Problem of Maintaining Large GraphQL Files

When I took over the Deliany project, one of the immediate challenges was how it exposed almost all data to GraphQL. Nearly every Ecto schema had a corresponding GraphQL type. While this isn't a dealbreaker for small projects, it becomes a maintenance nightmare as the application grows across multiple layers.

Consider a `books` table in PostgreSQL: we have a `Book` Ecto schema, which leads to a `BookType` in Absinthe. Absinthe then generates a `type Book` in `schema.graphql`, which is passed to the frontend to generate a TypeScript `type Book`. To make matters worse, tools like the Apollo GraphQL generator often produce many `Maybe<Type>` fields—which is reasonable but often too loose for strict application logic. This often forces us to generate yet another validated type, like `SafeBook`, with stricter typing for forms and displays. I felt there was significant duplication in storing and syncing schemas across Absinthe, the backend SDL, frontend types, and TypeScript, often requiring custom CI scripts to keep them in sync.

Another issue is the volume of data. Exposing everything might speed up development initially, especially with a frontend-heavy team, but pushing too much business logic to the frontend risks data leaks, wastes network traffic, and leads to stale data. Reorganizing these types, auditing their usage, and moving logic back to the backend is high-impact work that optimizes both the application and the workflow.

During this reorganization, I noticed that even harmless changes—like moving or renaming files—caused massive, noisy diffs in the `schema.graphql` file. This happens because Absinthe renders the SDL based on the order in which types are encountered. I wanted the generated schema to be deterministic and resistant to these structural changes. While Absinthe doesn't support sorted output out of the box, the good news is that it allows for custom rendering. The rendering logic is quite straightforward, as seen in the [Absinthe source code](https://github.com/absinthe-graphql/absinthe/blob/v1.6.8/lib/absinthe/schema/notation/sdl_render.ex). To make the schema easier to diff, we can sort types and fields by their names before rendering.

```elixir
defp render(%Blueprint{} = bp, _) do
    # ...
    all_type_definitions =
      type_definitions
      |> Enum.reject(&(&1.__struct__ == Blueprint.Schema.SchemaDeclaration))
      |> Enum.sort_by(fn
        %{name: "RootQueryType"} ->
          "0"

        %{name: "RootMutationType"} ->
          "1"

        %{name: "RootSubscriptionType"} ->
          "2"

        %Absinthe.Blueprint.Schema.ScalarTypeDefinition{name: name} ->
          "3#{Absinthe.Blueprint.Schema.ScalarTypeDefinition}-#{name}"

        %Absinthe.Blueprint.Schema.EnumTypeDefinition{name: name} ->
          "4#{Absinthe.Blueprint.Schema.EnumTypeDefinition}-#{name}"

        %m{name: name} ->
          "#{m}-#{name}"
      end)

    # ...
  end
```

I've shared my full version here: [Utils.Absinthe.Schema.Notation.SDL.MyRender](https://gist.github.com/anvox/d5b4777a3092b8b9a243130f014b7b74).

How do we tell Absinthe to use this custom renderer? We need to create a custom Mix task to generate the `schema.graphql` file. You can follow the pattern of the [standard Absinthe SDL task](https://github.com/absinthe-graphql/absinthe/blob/v1.5.5/lib/mix/tasks/absinthe.schema.sdl.ex) but swap in the custom renderer:

```elixir
def generate_schema(%Options{schema: schema}) do
  # ...
  case Absinthe.Pipeline.run(schema.__absinthe_blueprint__(), pipeline) do
    {:ok, blueprint, _phases} ->
      {:ok,
       Utils.Absinthe.Schema.Notation.SDL.MyRender.inspect(blueprint, %{pretty: true})}

    _ ->
      {:error, "Failed to render schema"}
  end
end
```

### TL;DR

While some features like deterministic SDL sorting seem obvious but are missing from the Absinthe core, the library is flexible enough to be patched for your specific needs. I'm generally not a fan of maintaining custom patches, but compared to the pain of managing noisy diffs across multiple 4,000-line GraphQL schemas, this small customization is a price well worth paying.
