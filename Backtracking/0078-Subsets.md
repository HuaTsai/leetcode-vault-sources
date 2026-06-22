---
leetcode-id: 78
difficulty: medium
tags:
  - backtracking
  - bit-manipulation
  - grind-169
  - neetcode-150
memo: 迭代法從空集合出發，每遇一數就複製現有全部子集再附加該數；另可回溯（每位選/不選）或位元遮罩（mask 0~2^n-1 逐位取），皆 O(n·2^n)
dg-publish: true
---

## Problem Description

Given an integer array `nums` of **unique** elements, return *all possible* *subsets* *(the power set)*.

The solution set **must not** contain duplicate subsets. Return the solution in **any order**.

All the numbers of `nums` are **unique**.

## Solution

```cpp
// Time: O(n2^n)
// Space: O(n2^n) / O(1)
vector<vector<int>> subsets(vector<int> &nums) {
  vector<vector<int>> ans{{}};
  for (auto n : nums) {
    int sz = ans.size();
    for (int i = 0; i < sz; ++i) {
      // C++17 後，emplace_back 回傳最後一項的 reference
      ans.emplace_back(ans[i]).emplace_back(n);
    }
  }
  return ans;
}
```

時間複雜度為 $O(n2^n)$，因為總共要迭代 $1 + 2^1 + ... + 2^{n-1} = 2^n - 1$ 個子集合，且需要考慮複製每個子集合的時間，子集合平均長度為 n/2

空間複雜度為 $O(n2^n)$，為考慮輸出空間，否則最佳額外空間可為 $O(1)$

```cpp
// Time: O(n2^n)
// Space: O(n2^n) / O(n)
vector<vector<int>> ans;
vector<int> cur;

void bt(const vector<int> &nums, int i) {
  if (i == nums.size()) {
    ans.push_back(cur);
    return;
  }
  bt(nums, i + 1);
  cur.push_back(nums[i]);
  bt(nums, i + 1);
  cur.pop_back();
}

vector<vector<int>> subsets(vector<int> &nums) {
  bt(nums, 0);
  return ans;
}
```

其中額外空間 $O(n)$ 為遞迴堆疊空間

```cpp
// Time: O(n2^n)
// Space: O(n2^n) / O(1)
vector<vector<int>> subsets(vector<int> &nums) {
  vector<vector<int>> ans;
  int n = nums.size();
  int total = 1 << n;
  for (int mask = 0; mask < total; ++mask) {
    vector<int> cur;
    for (int i = 0; i < n; ++i) {
      if (mask & (1 << i)) {
        cur.push_back(nums[i]);
      }
    }
    ans.push_back(cur);
  }
  return ans;
}
```
