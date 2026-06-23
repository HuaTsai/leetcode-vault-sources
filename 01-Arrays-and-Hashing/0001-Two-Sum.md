---
leetcode-id: 1
difficulty: easy
tags:
  - hash
  - grind-169
  - neetcode-150
memo: 一趟掃描，hash map 存「值→索引」；先查 target-nums[i] 是否已存在再把自己存入，確保不重用同一元素
dg-publish: true
---

## Problem Description

Given an array of integers `nums` and an integer `target`, return indices of the two numbers such that they add up to `target`.

You may assume that each input would have exactly one solution, and you may not use the same element twice.

You can return the answer in any order.

## Solution

```cpp
vector<int> twoSum(vector<int> &nums, int target) {
  unordered_map<int, int> mp;
  for (int i = 0; i < nums.size(); ++i) {
    int x = target - nums[i];
    if (mp.contains(x)) {
      return {mp[x], i};
    }
    mp[nums[i]] = i;
  }
  return {};
}
```
