---
leetcode-id: 347
difficulty: medium
tags:
  - hash
  - heap
  - neetcode-150
memo: hash 統計頻率後，用大小為 k 的 min-heap 取 top-k 達 O(nlogk)；進階用桶排序（頻率當索引、從高頻桶往回收集）達 O(n)
dg-publish: true
---

## Problem Description

Given an integer array `nums` and an integer `k`, return _the_ `k` _most frequent elements_. You may return the answer in **any order**.

## Solution

核心觀念：先用 hash map 統計每個數字的出現頻率，再從頻率中取出前 `k` 大。取 top-k 的手段很多，取捨在時間複雜度與常數：全排序 O(n log n)、堆／有序集合 O(n log k)、桶排序 O(n)。**以下所有方法第一步都是統計頻率**，差別只在「怎麼取出前 k 大」。

### 方法一：排序法 — O(n log n)／O(n)

把 (數字, 頻率) 存進 vector，依頻率由大到小排序後取前 `k` 個。最直觀，但帶了完整排序的 log n。

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

> [!tip]
> 只要前 `k` 個，不必全排：把 `ranges::sort` 換成 `partial_sort`（只排出前 k 個）即可降到 O(n log k)。

### 方法二：Min-Heap（最佳實務）— O(n log k)／O(n)

維護大小為 `k` 的最小堆，堆頂永遠是目前第 `k` 大的頻率；每來一個就 push，超過 `k` 就 pop 掉最小的，最後留下的就是 top-k。實務上這是最推薦的做法。

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

> [!note]
> C++ 的 `priority_queue` 無法用 iterator 遍歷，只能反覆 `top()` / `pop()` 把元素取出。

### 方法三：有序 Set — O(n log k)／O(n)

用 `set<pair<freq, num>>` 自動維持有序、保持大小為 `k`，超過就刪掉最小的。複雜度與 heap 相同，但紅黑樹常數較大，一般不如 heap 快——有趣的是**沒開優化時 `set` 反而較快**。

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

### 方法四：桶排序（理論最優）— O(n)／O(n)

桶的索引 = 出現頻率，桶的內容 = 該頻率的數字。從最高頻率的桶往回收集，湊滿 `k` 個即回傳，完全避開排序／堆的 log。

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

> [!important]
> 桶數量上限開「數字總數 **+ 1**」：最極端是全部都是同一個數字（頻率 = n），加一是為了讓索引含 0、計算方便。又因同一頻率可能對應多個數字，桶要用 `vector<vector<int>>` 而非 `vector<int>`。

### 方法五：PBDS priority_queue（可改值，邊掃邊維護 k）— O(n log k)／O(n)

用 `__gnu_pbds` 的 pairing heap，支援 `modify` 就地更新某數字的頻率。因為頻繁 modify，這裡反而表現最差，屬「能做但不划算」的示範。

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

### 方法六：PBDS priority_queue（全入再取 k）— O(n log n)／O(n)

所有元素先入最大堆、邊掃邊 `modify` 累加頻率，最後連續取 `k` 次堆頂。

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

- [[3224-Sliding-Window-Mode]] — 滑動視窗內動態維護眾數，延伸「頻率統計 + 取最高頻」。
- [[0692-Top-K-Frequent-Words]] — 幾乎相同，改成字串且同頻要字典序，heap 比較器需自訂。
- [[0451-Sort-Characters-By-Frequency]] — 依頻率重排字元，桶排序同款套路。
