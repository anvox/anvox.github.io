---
layout: post
title: Recall on Bloom Filter - Probabilistic data structures
date: 2026-04-17 12:30 +0700
categories: algorithm probabilistic-data-structures bloom-filter cuckoo-filter
---

Welcome to the land of probability. The name probabilistic data structures may hint about this category's outcome already: data structures with "maybe" or "% correct" in their answers. "Is it useless?" - it's my first thought when heard about them. Among vast of deterministic data structures, array, hash map, linked list... why do we need one, which couldn't give an exact answer? But after dipped into the probabilistic world, I found that when we need them, they are the right tool for the job. Let's move on. 

### We're going to resending some millions emails, at once

In my previous project, we had to send emails to all users at specific times. It wasn't a big deal, except for the fact that at that time, our system - an rails app - is not stable enough. Sometimes, the system would crash or hang, leaving emails undelivered. Our implemetation was quite simple, one master job get all emails subscribed to a campaign and sent them in parallel single jobs. On problem, we have to resend emails, in easiest way, by retrying email sending master job. Then each single email job has to check if the email has already been sent before sending it. This is a must-have feature to avoid sending duplicate emails as in our business, duplicate emails is worse than not sending them at all. So at the time of sending, millions db checks are performed to determine if an email has already been sent. This is not efficient, and the system becomes slow as the number of emails increases. We need to check email fast and efficiently. Here come the bloom filter.

Instead of using a database to store sent emails, we can use a bloom filter to store email hashes. Bloom filters could tell us if an email has already been sent without a database lookup. The bloom filter has a false positive rate, but in our case, we just simple fallback to a db lookup. The goal is not to replace the database, but to improve performance. We implemented a simple bloom filter using ruby and redis, then use it for years without any issues.

### What's is a bloom filter

Technically, a bloom filter is a list of N bits and M hash functions. Each hash function maps an email to a bit in those N bits. When an email sent, we calculate and set the corresponding M bits in the bloom filter. When checking an email, we calculate the M bits and check if all are set. If all bits are set, the email is likely already in the filter (false positive). If any bit is not set, the email is definitely not in the filter (false negative). With performance on [redis bitmaps](redis.io/docs/latest/develop/data-types/bitmaps/) and hash functions, this implement outperforms db check. 

#### Let's try a fiction example. 

A 10 bits bloom filter with 2 hash functions, `h1 = countif('a')` and `h2 = countif('b')`, which counts number of `'a'` and '`b'` in the input. 

```
bf = bx0000000000

# Adding 'apple' to bloom filter
h1a = h1('apple') = 1
h2a = h2('apple') = 0
bf = bx0000000011

# Adding 'banana' to bloom filter
h1b = h1('banana') = 3
h2b = h2('banana') = 1
bf = bx0000001011
```

Checking if `apple` and `banana` are in the filter is quite straightforward, recalculating the hash values and checking the corresponding bits in the bloom filter.

When checking `guava`, with `h1g = 2` and `h2g = 0`. We could quickly see `bf` has bit 2 unset (0), so `guava` is not in the filter.
```
h1g = h1('guava') = 2
h2g = h2('guava') = 0
bf = bx0000001011
```

When checking `papaya`, with `h1p = 3` and `h2p = 0`, both are set, from `h1b` of `banana` and `h2a` from `apple`.  So `papaya` is likely already in the filter as a false positive.
```
h1p = h1('papaya') = 3
h2p = h2('papaya') = 0
bf = bx0000001011
```

#### Pros and cons

One of advantages of bloom filters is their space efficiency, single bit per item, regardless of the number of items in the filter. And performance O(1) for both insertion and lookup. But there are some trade offs, which most significant is no deletion supports. Inserting to bloom filter is one way road. So there's many different variations of bloom filters that support deletion, but they all come with their own trade offs. One of them is cuckoo filter.

### Cuckoo filter

Everytime googling about bloom filer, it'll mention to cuckoo filter like an improvement over bloom filter. Cuckoo filter supports deletion and has better space efficiency than bloom filter, but with a price. Besides the complexity, cuckoo filter comes with slower and possibility of failure insertion. 

The filter now is a list of buckets, each bucket contains a list of items. And a hash function, which map an item to a bucket. 
To add an item to filter, we need to calculate 2 positions to insert the item. If both positions - buckets are full, we'll kick out an item from one of the buckets for new item insertion. Then use the same process to insert the kicked item. You'll notice when getting full, it may keep kicking out items indefinitely. So we have to limit the number of kicks to avoid this. To check if an item is in the filter, we need to check both positions - buckets.

There are some optimizations. First, we don't store whole input, just a small fingerprint of the item. Second, use a function to calculate secondary position from primary position and fingerprint to avoid rerun hash function. Let's move to an oversimplified example.

#### A fiction example

We use a filter with 8 buckets, with each bucket containing 1 item, to make confliction more frequent. In real life, we use `m` buckets with `b` items per bucket, for capacity of `m * b` items. Use a simple modulo function as the hash function.

```
# Filter:
0 - 
1 - 
2 - 
3 - 
4 - 
5 - 
6 - 
7 - 
```

Similar to previous examples, use `h = countif('a')` as the hash function. And `f = countif('a') + countif('b')` as the fingerprint function.

📦 Adding `apple`
```
i1 = h('apple') % 8 = 1 % 8 = 1
f =  f('apple') = 1

# Added `apple` fingerprint `1` to bucket 1:
0 - 
1 - 1
2 - 
3 - 
4 - 
5 - 
6 - 
7 - 
```

Adding `banana`
```
i1 = h('banana') % 8 = 3 % 8 = 3
f = f('banana') = 4

# Added `banana`:
0 - 
1 - 1
2 - 
3 - 4
4 - 
5 - 
6 - 
7 - 
```

Adding `papaya`
```
i1 = h('papaya') % 8 = 3
f = f('papaya') = 3

# Added `papaya`, but fingerprint `4` is kicked out:
0 - 
1 - 1
2 - 
3 - 3
4 - 
5 - 
6 - 
7 - 

# Finding new slot for fingerprint `4`
i2 = 3 XOR h2(4) = 3 XOR 4 = 7

# Added back fingerprint `4`
0 - 
1 - 1
2 - 
3 - 3
4 - 
5 - 
6 - 
7 - 4
```

🔎 Let's test if items in filter
Test `apple`
```
f = f('apple') = 1
i1 = h('apple') % 8 = 1
i2 = 1 XOR h2(1) = 1 XOR 1 = 0

Filter[1] == 1
Filter[0] == null
# So `apple` are in the filter
```

Test `banana`
```
f = f('banana') = 4
i1 = h('banana') % 8 = 3
i2 = 3 XOR h2(4) = 3 XOR 4 = 7

Filter[3] == 3 != 4 
Filter[7] == 4 
# So `banana` are in the filter
```

Test `papaya`
```
f = f('papaya') = 3
i1 = h('papaya') % 8 = 3
i2 = 3 XOR h2(3) = 3 XOR 3 = 0

Filter[3] == 3 
Filter[0] == null != 3
# So `papaya` are in the filter
```

Test `bamboo`
```
f = f('bamboo') = 2
i1 = h('bamboo') % 8 = 1
i2 = 1 XOR h2(2) = 1 XOR 2 = 3

Filter[1] == 1 != 2
Filter[3] == 3 != 2
# So `bamboo` are in the filter
```

Remove `banana` by finding it first:
```
i1 = h('banana') % 8 = 3
f1 = f('banana') = 4
i2 = 3 XOR h2(4) = 3 XOR 4 = 7

Filter[3] == 3 != 4 
Filter[7] == 4 
# So `banana` are in the filter at positions 7

# Removed `banana`
0 - 
1 - 1
2 - 
3 - 3
4 - 
5 - 
6 - 
7 -
```

⚠️ In this example, we use `h2` return same input value for simplicity, but in practice, `h2` should be a simple hash function. As we usually use small value for fingerprint, reusing them in i2 calculation causes poor distribution.

#### Redis support

At the time I write this, redis supports bloom filter and cuckoo filter already, just install and use them when you need.
[https://redis.io/docs/latest/develop/data-types/probabilistic/cuckoo-filter/](https://redis.io/docs/latest/develop/data-types/probabilistic/cuckoo-filter/)

### tl;dr

There are something on earth called probabilistic data structures which sounds useless but are actually very useful in some cases. And [redis](https://redis.io/docs/latest/develop/data-types/probabilistic/) already supports them well.
