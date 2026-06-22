---
leetcode-id: 703
difficulty: easy
tags:
  - binary-search-tree
  - policy-based-data-structures
  - heap
  - neetcode-150
memo: 基礎最小堆應用
dg-publish: true
---

## Problem Description

You are part of a university admissions office and need to keep track of the `kth` highest test score from applicants in real-time. This helps to determine cut-off marks for interviews and admissions dynamically as new applicants submit their scores.

You are tasked to implement a class which, for a given integer `k`, maintains a stream of test scores and continuously returns the `k`th highest test score **after** a new score has been submitted. More specifically, we are looking for the `k`th highest score in the sorted list of all scores.

Implement the `KthLargest` class:

- `KthLargest(int k, int[] nums)` Initializes the object with the integer `k` and the stream of test scores `nums`.
- `int add(int val)` Adds a new test score `val` to the stream and returns the element representing the `kth` largest element in the pool of test scores so far.

## Solution

使用 `multiset` 解

```cpp
// Time: O(nlog(k)) 建構子 / O(log(k)) 新增函數
// Space: O(k)
multiset<int> scores;
int k;

KthLargest(int k, vector<int> &nums) : k(k) {
  for (int n : nums) {
    scores.insert(n);
    if (scores.size() > k) {
      scores.erase(scores.begin());
    }
  }
}

int add(int val) {
  scores.insert(val);
  if (scores.size() > k) {
    scores.erase(scores.begin());
  }
  return *scores.begin();
}
```

最大 K 問題也對應到 min heap 的資料結構，一般使用 heap 開銷稍小

注意 `priority_queue` 預設為 max heap 所以要額外傳 template 參數

```cpp
// Time: O(nlog(k)) 建構子 / O(log(k)) 新增函數
// Space: O(k)
priority_queue<int, vector<int>, greater<>> scores;
int k;

KthLargest(int k, vector<int> &nums) : k(k) {
  for (int n : nums) {
    scores.push(n);
    if (scores.size() > k) {
      scores.pop();
    }
  }
}

int add(int val) {
  scores.push(val);
  if (scores.size() > k) {
    scores.pop();
  }
  return scores.top();
}
```

最後有 pbds 的解法

```cpp
#include <ext/pb_ds/assoc_container.hpp>
#include <ext/pb_ds/tree_policy.hpp>

using Set = __gnu_pbds::tree<pair<int, int>, __gnu_pbds::null_type,
                             std::greater<>, __gnu_pbds::rb_tree_tag,
                             __gnu_pbds::tree_order_statistics_node_update>;

Set scores;
int k;

KthLargest(int k, vector<int> &nums) : k(k) {
  for (int i = 0; i < nums.size(); ++i) {
    scores.insert({nums[i], i});
  }
}

int add(int val) {
  scores.insert({val, scores.size()});
  return scores.find_by_order(k - 1)->first;
}
```

此解法因為沒有刪除不必要元素，所以並沒有比較快，但是可以參考 `find_by_order` 的使用方法

處理重複元素建議加上索引值而不是用 `std::greater_equal`
