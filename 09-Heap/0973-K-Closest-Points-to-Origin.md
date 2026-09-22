---
leetcode-id: 973
difficulty: medium
tags:
  - heap
  - array
  - geometry
  - kd-tree
  - sort
  - grind-169
  - neetcode-150
memo: 維護大小 k 的 max-heap 存｛距離平方，索引｝，比堆頂近才進堆；追問 O(n) 就是 nth_element 做 quickselect
dg-publish: true
---

## Problem Description

Given an array of `points` where `points[i] = [xi, yi]` represents a point on the X-Y plane and an integer `k`, return the `k` closest points to the origin `(0, 0)`.

The distance between two points on the X-Y plane is the Euclidean distance (i.e., `√(x1 - x2)^2 + (y1 - y2)^2`).

You may return the answer in any order. The answer is guaranteed to be unique (except for the order that it is in).

## Solution

「前 k 近」不需要完整排序。用一個**大小為 k 的 max-heap**裝目前最近的 k 個點，堆頂就是這 k 個裡最遠的：新點只要比堆頂近，就把堆頂踢掉換它進來，掃完一輪堆裡剩的就是答案。距離比大小不必開根號，直接比 `x² + y²`（座標 ≤ 10⁴，平方和 ≤ 2×10⁸，`int` 裝得下）。

> [!important]
> 「前 k 小」用 **max-heap**、「前 k 大」用 **min-heap**：堆頂永遠是候選集合裡「最該被淘汰」的那個。

### 方法一：大小 k 的 max-heap — O(n log k)／O(k)

```cpp
// Time: O(n log k)
// Space: O(k)
class Solution {
 public:
  vector<vector<int>> kClosest(vector<vector<int>>& points, int k) {
    priority_queue<pair<int, int>> pq;  // {dist², index}，堆頂是候選裡最遠的
    for (int i = 0; i < points.size(); ++i) {
      int d = points[i][0] * points[i][0] + points[i][1] * points[i][1];
      if (pq.size() < k) {
        pq.emplace(d, i);
      } else if (d < pq.top().first) {  // 比堆頂近才值得換
        pq.pop();
        pq.emplace(d, i);
      }
    }
    vector<vector<int>> ans;
    ans.reserve(k);
    while (!pq.empty()) {
      ans.push_back(points[pq.top().second]);
      pq.pop();
    }
    return ans;
  }
};
```

> [!tip]
> 常見寫法是每個點都先 `push` 再 `if (size > k) pop`，正確但每個點都付兩次 O(log k) 的 sift。隨機資料下第 i 個點進得了前 k 名的機率只有 k／i，期望插入次數約 k·ln(n／k)，所以**先跟堆頂比、贏了才換**，絕大多數點一次比較就淘汰。n = 200000、k = 10 實測從 2.3 ms 降到 0.34 ms。

> [!note]
> `pair` 的預設 `less<>` 先比 `first`，所以 `priority_queue<pair<int, int>>` 直接就是照距離的 max-heap，不需要另寫 comparator 再 `decltype(cmp)`；同距時會再比索引，對答案沒影響。堆裡存索引而不是點本身，省下複製 `vector<int>` 的開銷。

### 方法二：nth_element（quickselect） — 平均 O(n)／O(n)

面試追問「能不能比 O(n log k) 更好」的標準答案：不需要前 k 名排好序，只要把第 k 小的距離放到正確位置、左邊全是比它小的即可，這正是 `nth_element` 做的事（quickselect，平均 O(n)、最壞 O(n²)，libstdc++ 用 introselect 退化時改 heapselect 保 O(n log n)）。

```cpp
// Time: O(n) 平均
// Space: O(n)
class Solution {
 public:
  vector<vector<int>> kClosest(vector<vector<int>>& points, int k) {
    vector<pair<int, int>> d(points.size());  // {dist², index}
    for (int i = 0; i < points.size(); ++i) {
      d[i] = {points[i][0] * points[i][0] + points[i][1] * points[i][1], i};
    }
    ranges::nth_element(d, d.begin() + k);
    vector<vector<int>> ans;
    ans.reserve(k);
    for (int i = 0; i < k; ++i) {
      ans.push_back(points[d[i].second]);
    }
    return ans;
  }
};
```

> [!warning]
> 更短的寫法是直接 `ranges::nth_element(points, points.begin() + k, {}, dist2)` 然後 `points.resize(k)`，**k 小時反而比 heap 慢**：它 swap 的是整個內層 `vector<int>`（三個指標），投影函式每次比較都重算距離，而且內層 vector 散在堆上、cache 不友善。先把 {dist²，index} 抽成 8 bytes 的 `pair` 陣列再選，距離只算一次、搬動也便宜，才發揮得出 O(n) 的優勢。

n = 200000 的實測（wall clock，5 次取最佳，`-O2`）：

```txt
k        heap push再pop   heap 先比堆頂   nth_element 在 points   nth_element 在 pair
10         2.30 ms          0.34 ms          3.08 ms                 1.61 ms
1000       4.81 ms          0.84 ms          5.34 ms                 1.68 ms
100000    21.4  ms         20.4  ms          7.46 ms                 4.55 ms
```

> [!note]
> k ≪ n 時「先比堆頂」的 heap 最快，因為幾乎每個點都只做一次比較；k 接近 n 時 heap 退化成 O(n log n)，`nth_element` 才明顯勝出。heap 另一個優點是**串流友善**：資料一筆一筆來也能維護，`nth_element` 得先拿到全部。

## Related Problems

- [[0703-Kth-Largest-Element-in-a-Stream]] — 同樣是大小 k 的 heap，堆頂就是第 k 名
- [[0215-Kth-Largest-Element-in-an-Array]] — heap 與 nth_element 兩條路的原型題
- [[0347-Top-K-Frequent-Elements]] — 前 k 名的另一種鍵值（頻率），同樣可用 heap 或桶
- [[0658-Find-K-Closest-Elements]] — 「最近 k 個」但陣列已排序，改用二分搜尋找視窗起點
