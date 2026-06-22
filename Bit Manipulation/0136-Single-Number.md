---
leetcode-id: 136
difficulty: easy
tags:
  - bit-manipulation
  - grind-169
  - neetcode-150
memo: 全部元素 XOR 累加；關鍵是 a^a=0、a^0=a，成對者互相抵銷只剩單一數，O(1) 空間
dg-publish: true
---

## Problem Description

Given a **non-empty** array of integers `nums`, every element appears *twice* except for one. Find that single one.

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
