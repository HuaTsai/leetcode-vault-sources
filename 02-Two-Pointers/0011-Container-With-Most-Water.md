---
leetcode-id: 11
difficulty: medium
tags:
  - two-pointers
  - array
  - greedy
  - grind-169
  - neetcode-150
memo:
dg-publish: true
---

## Problem Description

You are given an integer array `height` of length `n`. There are `n` vertical lines drawn such that the two endpoints of the `ith` line are `(i, 0)` and `(i, height[i])`.

Find two lines that together with the x-axis form a container, such that the container contains the most water.

Return the maximum amount of water a container can store.

Notice that you may not slant the container.

## Solution

這題需要左右兩側的索引往中間靠攏，靠攏的條件是下個索引比前個索引的值高，重點需要說明此方法的正確性

面積公式為 `Area = min(h[l], h[r]) * (r - l)`，

```cpp
// Time: O()
// Space: O()
```

## Related Problems

