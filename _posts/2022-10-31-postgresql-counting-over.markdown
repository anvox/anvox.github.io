---
layout: post
title: Postgresql counting over
date: 2022-10-31 12:19 +0700
categories: postgresql count count-over date_trunc til
---

## Truncate datetime

```sql
SELECT DISTINCT
       date_trunc('minute', "when") AS minute
     , count(*) OVER (ORDER BY date_trunc('minute', "when")) AS running_ct
FROM   mytable
ORDER  BY 1;
```

## Extract parts of datetime

Ref [date_part](https://www.geeksforgeeks.org/postgresql-date_part-function/)

```sql
SELECT date_part('year', "inserted_at") FROM users
```
