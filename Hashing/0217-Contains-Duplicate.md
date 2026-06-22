---
leetcode-id: 217
difficulty: easy
tags:
  - hash
  - grind-169
  - neetcode-150
memo: 邊掃描邊用 unordered_set：先查再插，命中已存在元素即回 true，一趟 O(n)
dg-publish: true
---

## Problem Description

Given an integer array `nums`, return `true` if any value appears at least twice in the array, and return `false` if every element is distinct.

## Solution

```cpp
bool containsDuplicate(vector<int> &nums) {
  unordered_set<int> st;
  for (int i : nums) {
    if (st.contains(i)) {
      return true;
    }
    st.insert(i);
  }
  return false;
}
```
