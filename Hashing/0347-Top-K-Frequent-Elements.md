---
leetcode-id: 347
difficulty: medium
tags:
  - hash
  - heap
  - neetcode-150
memo: hash + min-heap 應用
dg-publish: true
---

## Problem Description

Given an integer array `nums` and an integer `k`, return *the* `k` *most frequent elements*. You may return the answer in **any order**.

## Solution

以下做法一到三共同部份是要先計算出項目出現頻率

初步撰寫，將項目與頻率存入 vector 後做排序

```cpp
// Time: O(nlog(n))
// Space: O(n)
vector<int> topKFrequent(vector<int> &nums, int k) {
  unordered_map<int, int> mp;
  for (int n : nums) {
    ++mp[n];
  }
  vector<pair<int, int>> x;
  for (auto [k, v] : mp) {
    x.push_back({k, v});
  }
  ranges::sort(x, [](auto a, auto b) { return a.second > b.second; });
  vector<int> ans;
  for (int i = 0; i < k; ++i) {
    ans.push_back(x[i].first);
  }
  return ans;
}
```

**可以將 `sort` 修改成 C++17 後的 `partial_sort` 達到  O(nlog(k))**

第二種做法，使用 min heap 或 red-black tree 將求 top k 的時間複雜度拉低

```cpp
// Time: O(nlog(k))
// Space: O(n)
vector<int> topKFrequent(vector<int> &nums, int k) {
  unordered_map<int, int> mp;
  for (int n : nums) {
    ++mp[n];
  }
  priority_queue<pair<int, int>, vector<pair<int, int>>, greater<>> pq;
  for (auto [num, freq] : mp) {
    pq.push({freq, num});
    if (pq.size() > k) {
      pq.pop();
    }
  }
  vector<int> ans;
  while (pq.size()) {
    ans.push_back(pq.top().second);
    pq.pop();
  }
  return ans;
}
```

**注意 C++ 沒有辦法用 iterator 遍歷 priority queue**

單純的 `set` 做法，時間複雜度與 heap 相同，但維持紅黑樹的成本比 heap 還高，所以不推薦，統一用 heap 會比較快

值得注意的是沒有開優化時反而 `set` 比較快

```cpp
// Time: O(nlog(k))
// Space: O(n)
vector<int> topKFrequent(vector<int> &nums, int k) {
  unordered_map<int, int> mp;
  for (int n : nums) {
    ++mp[n];
  }
  set<pair<int, int>> st;
  for (auto [num, freq] : mp) {
    st.insert({freq, num});
    if (st.size() > k) {
      st.erase(st.begin());
    }
  }
  vector<int> ans;
  for (auto [freq, num] : st) {
    ans.push_back(num);
  }
  return ans;
}
```

第三種做法使用桶排序，桶子編號是出現頻率，內容是數字，會快很多

```cpp
// Time: O(n)
// Space: O(n)
vector<int> topKFrequent(vector<int> &nums, int k) {
  unordered_map<int, int> mp;
  for (int n : nums) {
    ++mp[n];
  }
  vector<vector<int>> buckets(nums.size() + 1);
  for (auto [num, freq] : mp) {
    buckets[freq].push_back(num);
  }
  vector<int> ans;
  for (int i = nums.size(); i >= 0; --i) {
    for (auto n : buckets[i]) {
      ans.push_back(n);
      if (ans.size() == k) {
        return ans;
      }
    }
  }
  return {};
}
```

關鍵點是桶子數量上限可以開輸入**數字數量 + 1**，因為最極端就是全部都是相同的數字，加一是包含 0 方便計算

另外注意可能有重複情況，所以要開 `vector<vector<int>>` 而不是 `vector<int>`

第四種做法，使用 PBDS 裡的 priority queue，支援修改值，因為頻繁修改此處表現會最差

```cpp
// Time: O(nlog(k))
// Space: O(n)
using pq_type = __gnu_pbds::priority_queue<pair<int, int>, greater<>,
                                           __gnu_pbds::pairing_heap_tag>;
vector<int> topKFrequent(vector<int> &nums, int k) {
  pq_type pq;
  unordered_map<int, pq_type::point_iterator> iterators;
  unordered_map<int, int> count;

  for (int num : nums) {
    count[num]++;
    int freq = count[num];

    if (iterators.contains(num)) {
      pq.modify(iterators[num], {freq, num});
    } else {
      iterators[num] = pq.push({freq, num});
      if (pq.size() > k) {
        int removed = pq.top().second;  // 使用 min-heap 因為打算移掉最低頻率
        iterators.erase(removed);
        pq.pop();
      }
    }
  }

  vector<int> result;
  while (!pq.empty()) {
    result.push_back(pq.top().second);
    pq.pop();
  }
  return result;
}
```

```cpp
// Time: O(nlog(n))
// Space: O(n)
using pq_type = __gnu_pbds::priority_queue<pair<int, int>, less<>,
                                           __gnu_pbds::pairing_heap_tag>;
vector<int> topKFrequent(vector<int> &nums, int k) {
  pq_type pq;
  unordered_map<int, pq_type::point_iterator> iterators;

  for (int num : nums) {
    if (iterators.contains(num)) {
      pq.modify(iterators[num], {iterators[num]->first + 1, num});
    } else {
      iterators[num] = pq.push({1, num});
    }
  }

  // 一次建立完，然後從 heap 取頻率最大，所以用 max-heap
  vector<int> ans;
  while (k--) {
    ans.push_back(pq.top().second);
    pq.pop();
  }
  return ans;
}
```

| 版本                      | 時間複雜度 | 空間複雜度 | 原理                                                       |
| ------------------------- | ---------- | ---------- | ---------------------------------------------------------- |
| v1 排序法                 | O(nlog(n)) | O(n)       | 統計頻率後，將所有元素按頻率排序，取前 k 個                |
| v2 最小堆**（最佳選擇）** | O(nlog(k)) | O(n)       | 維護大小為 k 的最小堆，堆頂永遠是第 k 大，超過就踢掉最小的 |
| v3 Set                    | O(nlog(k)) | O(n)       | 用 set 自動維護有序，保持大小為 k，超過就刪除最小的        |
| v4 桶排序**（理論最佳）** | O(n)       | O(n)       | 用頻率當索引建立桶，從最高頻率的桶開始收集元素             |
| v5 PBDS-K                 | O(nlog(k)) | O(n)       | 邊遍歷邊用 modify 更新頻率，維護 k 大小最小堆              |
| v6 PBDS-All               | O(nlog(n)) | O(n)       | 所有元素入最大堆，邊遍歷邊 modify 更新頻率，最後取前 k 個  |

## Related Problems

[[3224-Sliding-Window-Mode]]
