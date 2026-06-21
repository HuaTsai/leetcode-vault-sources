---
leetcode-id: 190
difficulty: easy
tags:
  - bit-manipulation
  - grind-169
  - neetcode-150
solved-1st: 2025-12-25
solved-2nd:
solved-3rd:
memo: 基礎位元題目
dg-publish: true
---

## Problem Description

Reverse bits of a given 32 bits signed integer.

## Solution

```cpp
// Time: O(1)
// Space: O(1)
int reverseBits(int n) {
  int ans = 0;
  for (int i = 0; i < 32; ++i, n >>= 1) {
    ans = (ans << 1) | (n & 1);
  }
  return ans;
}
```
