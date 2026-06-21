---
leetcode-id: 239
difficulty: hard
tags:
  - monotonic-queue
  - grind-169
  - neetcode-150
solved-1st: 2025-12-16
solved-2nd:
solved-3rd:
memo: 使用單調佇列，熟悉算法實作
dg-publish: true
---

## Problem Description

You are given an array of integers `nums`, there is a sliding window of size `k` which is moving from the very left of the array to the very right. You can only see the `k` numbers in the window. Each time the sliding window moves right by one position.

Return _the max sliding window_.

## Solution

```cpp
// Time: O(n)
// Space: O(n) / O(k)
vector<int> maxSlidingWindow(vector<int> &nums, int k) {
  vector<int> ans;
  deque<pair<int, int>> q;
  for (int i = 0; i < nums.size(); ++i) {
    while (q.size() && q.back().first <= nums[i]) {
      q.pop_back();
    }

    q.push_back({nums[i], i});

    if (i - q.front().second >= k) {
      q.pop_front();
    }

    if (i >= k - 1) {
      ans.push_back(q.front().first);
    }
  }
  return ans;
}
```

## Related Problems

[[0239-Sliding-Window-Maximum]]
