---
layout: post
title: PostgreSQL Order by custom ordering
date: 2022-04-05 09:58 +0700
categories: til postgres postgresql
---

# Context

Sorting query by custom order, usually, by a state column. The most intuitive way is using `CASE WHEN`:

```sql
SELECT * FROM orders ORDER BY CASE
    WHEN orders.status = 'new' THEN 1
    WHEN orders.status = 'processing' THEN 2
    WHEN orders.status = 'done' THEN 3
    WHEN orders.status = 'cancelled' THEN 4
    WHEN orders.status = 'awaiting_confirmation' THEN 5
    ELSE 99
  END DESC;
```


Postgresql support `array_position()`, so we could write like this.

```sql
SELECT * FROM orders ORDER BY array_position(ARRAY['new', 'processing', 'done', 'cancelled', 'awaiting_confirmation']::varchar[], orders.status) DESC;
```

And finally, some suggests on stackoverflow:

```sql
SELECT *
FROM orders
JOIN unnest(ARRAY['new', 'processing', 'done', 'cancelled', 'awaiting_confirmation']) WITH ORDINALITY o(status, ord) USING (status)
ORDER BY o.ord DESC;
```

Which one is best, or better, when we should use which way? 2 factors to concern: readability and performance. About readability, `array_position` is prefered by devs, but `CASE WHEN` is voted by non-dev stackholders. About performance, a benchmark tells more than any word.

# Benchmark

I scripted to generate random 1mil records with random status to compare.

```sql
DROP TABLE testusers;
CREATE TABLE testusers(
    id BIGINT GENERATED ALWAYS AS IDENTITY,
    PRIMARY KEY(id),
    status VARCHAR(50) NOT NULL CHECK (status IN ('new', 'processing', 'done', 'cancelled', 'awaiting_confirmation'))
);

INSERT INTO testusers(status)
SELECT CASE
WHEN RANDOM() < 0.2 THEN 'new' 
WHEN RANDOM() < 0.4 THEN 'processing' 
WHEN RANDOM() < 0.6 THEN 'done' 
WHEN RANDOM() < 0.8 THEN 'cancelled' 
ELSE 'awaiting_confirmation' END
FROM generate_series(1, 1000000);


SELECT * FROM testusers ORDER BY array_position(ARRAY['new', 'processing', 'done', 'cancelled', 'awaiting_confirmation']::varchar[], testusers.status) ASC, testusers.id LIMIT 1000;

SELECT * FROM testusers ORDER BY CASE
    WHEN testusers.status = 'new' THEN 1
    WHEN testusers.status = 'processing' THEN 2
    WHEN testusers.status = 'done' THEN 3
    WHEN testusers.status = 'cancelled' THEN 4
    WHEN testusers.status = 'awaiting_confirmation' THEN 5
    ELSE 99
  END ASC, testusers.id LIMIT 1000;

SELECT *
FROM testusers
JOIN unnest(ARRAY['new', 'processing', 'done', 'cancelled', 'awaiting_confirmation']) WITH ORDINALITY o(status, ord) USING (status)
ORDER BY o.ord ASC, testusers.id LIMIT 1000;
```

# Conclusion

With test data, all 3 run with similar result, where `JOIN unnest` is slightly faster.<br />
But if we order result by addition column, `CASE` approach is fastest 🤔

So, try on your dataset to figure out which approach are fit to case.
