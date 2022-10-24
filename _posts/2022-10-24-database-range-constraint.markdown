---
layout: post
title: Database range constraint
date: 2022-10-24 10:17 +0700
categories: elixir postgresql constraint tsrange array
---

##

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE promotions
ADD CONSTRAINT overlapping_running_time
EXCLUDE USING gist (
  product_id WITH =,
  tsrange("start_at", "end_at", '[)') WITH &&
) WHERE ("status" = 'active');
```


```elixir
  |> exclusion_constraint(:start_at,
    name: :overlapping_running_time,
    message: "Discount of product #{product_id} is overlapped"
  )
```

🚨 Only `superuser` role has permission to `CREATE EXTENSION`.
