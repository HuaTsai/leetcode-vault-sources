---
leetcode-id: 217
difficulty: easy
tags:
  - hash
  - grind-169
  - neetcode-150
solved-1st: 2025-12-09
solved-2nd:
solved-3rd:
memo: 基礎雜湊表應用
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
