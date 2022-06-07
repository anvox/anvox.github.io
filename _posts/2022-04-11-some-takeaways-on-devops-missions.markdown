---
layout: post
title: Computer color is broken
date: 2022-06-07 09:17 +0700
categories: til color
---

Source: [Computer Color is Broken](https://www.youtube.com/watch?v=LKnqECcg6Gw)

# Problem

Sometimes, when render chart, to have new color for new item, I usually use average from items around. i.e. between <span style="background-color:#ff0000">red</span> bar and <span style="background-color:#00ff00">green</span> bar, I used the average transition color <span style="background-color:#7f7f00">#7f7f00 (127,127,0)</span> bar. You could see, <span style="background-color:#ff0000">#ff0000</span> <span style="background-color:#7f7f00">#7f7f00</span> <span style="background-color:#00ff00">#00ff00</span> it's weird, seems a bit darker than expected.

# Solution

## tl;dr

Use another lighter color: <span style="background-color:#b4b400">#b4b400 (180,180,0)</span> in this case.
<br />
<span style="background-color:#ff0000">#ff0000</span><span style="background-color:#b4b400">#b4b400</span><span style="background-color:#00ff00">#00ff00</span>

<span style="background-color:#ff0000">#ff0000</span><span style="background-color:#7f7f00">#7f7f00</span><span style="background-color:#00ff00">#00ff00</span>

How do we have the number `180`/`0xb4`:

```
r' = SQRT(r0^2 + r1^2) = SQRT((255^2) + (0^2)) = 180 = 0xb4
```

## Explaination

There are 2 factors of this problem.

First is the way our human eyes preceive the brightness. It's not straight, but in a roughly logarithmic scale. Technically, `rgb(0,0,0)` has 0 brightness, or 0 photons; `rgb(1,1,1)` - full brightness - has 100% photons. But `rgb(0.5, 0.5, 0.5)` has only 22% photons - 22% brightness. Even worse, `rgb(0.25, 0.25, 0.25)` has only 5% photons, not 25% brightness as our expected. The average doesn't apply here.

Second, the way CCD sensor records a pixel in computer. It's pure number. Moreover, it stores square root of the brightness, to save storage. By this, dark color datapoints - near 0 - are recorded more than bright color datapoints - near 1. This roughly imitate human vision. Then square back when display.

But, when we want to modify, remember, the rgb numbers we work on are square root, we must square back before take any action.
