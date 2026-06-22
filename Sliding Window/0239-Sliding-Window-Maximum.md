---
leetcode-id: 239
difficulty: hard
tags:
  - monotonic-queue
  - grind-169
  - neetcode-150
memo: 單調遞減 deque 存（值，索引）：入隊前彈掉隊尾所有 ≤ 新值者，隊首索引出窗就 pop_front，隊首恆為窗口最大值，O(n)
dg-publish: true
---

## Problem Description

You are given an array of integers `nums`, there is a sliding window of size `k` which is moving from the very left of the array to the very right. You can only see the `k` numbers in the window. Each time the sliding window moves right by one position.

Return *the max sliding window*.

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
