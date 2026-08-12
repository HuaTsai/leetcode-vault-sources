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

核心觀念：要判斷是否有重複，只需一邊掃描一邊記錄看過的數。用 `unordered_set` 做「先查再插」，一命中已存在的元素就能立刻回 `true`，不必等掃完。

### 方法一：Hash Set 一趟掃描 — O(n)／O(n)

```cpp
// Time: O(n)
// Space: O(n)
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

> [!tip]
> 早退（early return）：一發現重複就回傳，平均只掃到第一個碰撞為止，不用建完整個 set。

### 方法二：排序後比相鄰 — O(n log n)／O(1)

若不允許額外空間，可先排序再看相鄰是否相等；用時間與 in-place 排序換掉 hash 的空間。

```cpp
// Time: O(n log n)
// Space: O(1)
bool containsDuplicate(vector<int> &nums) {
  ranges::sort(nums);
  for (int i = 1; i < nums.size(); ++i) {
    if (nums[i] == nums[i - 1]) {
      return true;
    }
  }
  return false;
}
```

## Related Problems

- [[0242-Valid-Anagram]] — 同樣用計數／hash 判斷元素組成是否一致。
- [[0001-Two-Sum]] — 一趟 hash 邊掃邊查的入門姊妹題。
- [[0219-Contains-Duplicate-II]] — 進階加上索引距離限制 `k`，改用滑動視窗維護的 set。
