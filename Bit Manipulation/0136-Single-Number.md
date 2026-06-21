---
leetcode-id: 136
difficulty: easy
tags:
  - bit-manipulation
  - grind-169
  - neetcode-150
solved-1st: 2025-12-15
solved-2nd:
solved-3rd:
memo: 基礎異或應用
dg-publish: true
---

## Problem Description

Given a **non-empty** array of integers `nums`, every element appears _twice_ except for one. Find that single one.

You must implement a solution with a linear runtime complexity and use only constant extra space.

## Solution

```cpp
// Time: O(n)
// Space: O(1)
int singleNumber(vector<int> &nums) {
  return std::accumulate(nums.begin(), nums.end(), 0,
                         [](int res, int n) { return res ^ n; });
}
```
