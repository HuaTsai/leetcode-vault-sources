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

核心觀念：要找兩數之和為 `target`，與其兩兩枚舉，不如邊掃邊記。對每個 `nums[i]`，它需要的另一半固定是 `target - nums[i]`；用 hash map 存「看過的值 → 索引」，掃到 `i` 時先查另一半在不在 map 裡——在就是答案，不在就把自己存進去。把「查找另一半」從 O(n) 降到 O(1)，整體一趟就結束。

### 方法一：一趟 Hash Map — O(n)／O(n)

```cpp
// Time: O(n)
// Space: O(n)
vector<int> twoSum(vector<int> &nums, int target) {
  unordered_map<int, int> mp;  // 值 → 索引
  for (int i = 0; i < nums.size(); ++i) {
    int x = target - nums[i];
    if (mp.contains(x)) {
      return {mp[x], i};
    }
    mp[nums[i]] = i;  // 查不到才存自己
  }
  return {};
}
```

> [!tip]
> **先查 `target - nums[i]`，再把 `nums[i]` 存入**。順序反過來（先存再查）在 `target == 2 * nums[i]` 時會查到剛存進去的自己，等於重用同一個元素。

### 方法二：暴力枚舉 — O(n²)／O(1)

不需額外空間，但兩層迴圈慢；面試時當作對照組帶出 hash 解的優化動機。

```cpp
// Time: O(n^2)
// Space: O(1)
vector<int> twoSum(vector<int> &nums, int target) {
  for (int i = 0; i < nums.size(); ++i) {
    for (int j = i + 1; j < nums.size(); ++j) {
      if (nums[i] + nums[j] == target) {
        return {i, j};
      }
    }
  }
  return {};
}
```

## Related Problems

- [[0167-Two-Sum-II-Input-Array-Is-Sorted]] — 輸入已排序的版本，改用相向雙指針即可做到 O(1) 空間。
- [[0015-3Sum]] — 固定一個數後，對其餘元素做 Two Sum，是本題的直接延伸。
- [[0454-4Sum-II]] — 把四陣列拆成兩組各求和存 hash，再互相配對補數，本質仍是 Two Sum。
