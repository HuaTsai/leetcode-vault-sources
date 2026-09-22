---
leetcode-id: 1046
difficulty: easy
tags:
  - heap
  - array
  - neetcode-150
memo: 用 max-heap 反覆取兩顆最重相撞，差值推回堆；最後堆可能為空，回傳前要判 empty
dg-publish: true
---

## Problem Description

You are given an array of integers `stones` where `stones[i]` is the weight of the `ith` stone.

We are playing a game with the stones. On each turn, we choose the heaviest two stones and smash them together. Suppose the heaviest two stones have weights `x` and `y` with `x <= y`. The result of this smash is:

If `x == y`, both stones are destroyed, and
If `x != y`, the stone of weight `x` is destroyed, and the stone of weight `y` has new weight `y - x`.
At the end of the game, there is at most one stone left.

Return the weight of the last remaining stone. If there are no stones left, return `0`.

## Solution

每回合要的是「目前最重的兩顆」，而且撞完的殘餘會回到候選池裡再參與比較。這就是 max-heap 的典型場景：取堆頂兩次、把差值推回去，重複到剩不到兩顆。

> [!important]
> 迴圈結束時堆可能是**空的**（最後兩顆等重就會全滅），`top()` 對空堆是 UB，回傳前一定要判 `empty()`。

### 方法一：priority_queue — O(n log n)／O(n)

```cpp
// Time: O(n log n)
// Space: O(n)
class Solution {
 public:
  int lastStoneWeight(vector<int>& stones) {
    priority_queue<int> pq(stones.begin(), stones.end());
    while (pq.size() > 1) {
      int y = pq.top();
      pq.pop();
      int x = pq.top();
      pq.pop();
      if (y != x) {
        pq.push(y - x);
      }
    }
    return pq.empty() ? 0 : pq.top();
  }
};
```

> [!tip]
> 用 range 建構子 `priority_queue<int> pq(begin, end)` 建堆，底層是 `make_heap`（Floyd 建堆）O(n)；逐顆 `push` 則是 O(n log n)。這題 n ≤ 30 量不出差別，但這是寫 heap 的慣用手法。

### 方法二：原地 make_heap／pop_heap — O(n log n)／O(1)

`priority_queue` 底下就是 `make_heap`、`pop_heap`、`push_heap` 三個演算法，直接對輸入的 `stones` 操作就不用額外容器。`pop_heap` 會把堆頂搬到尾端，`back()` 取值後 `pop_back()`。

```cpp
// Time: O(n log n)
// Space: O(1)
class Solution {
 public:
  int lastStoneWeight(vector<int>& stones) {
    ranges::make_heap(stones);
    while (stones.size() > 1) {
      ranges::pop_heap(stones);
      int y = stones.back();
      stones.pop_back();
      ranges::pop_heap(stones);
      int x = stones.back();
      stones.pop_back();
      if (y != x) {
        stones.push_back(y - x);
        ranges::push_heap(stones);
      }
    }
    return stones.empty() ? 0 : stones[0];
  }
};
```

> [!note]
> 兩顆都 pop 完才 push 差值。若先 pop 一顆、push 差值、再 pop 第二顆，差值可能被當成「第二重」拿去撞，結果就錯了。

### 方法三：桶計數 — O(n + W)／O(W)

重量上限 W = 1000，開 1001 格的桶，從重到輕掃。同一重量的石頭兩兩抵消，剩單數才留一顆在手上；手上有石頭時就撞掉目前掃到的重量。

```txt
stones = [10, 8, 8, 8, 5, 2, 2]

w=10  手上空，cnt=1 奇數 → 手上 y=10
w=8   手上 y=10 > 8，撞 → y=2；2 ≤ 8 → 丟回 cnt[2]，手上清空，重掃 w=8
w=8   手上空，cnt=2 偶數 → 全滅
w=5   手上空，cnt=1 → y=5
w=2   手上 y=5 > 2，撞 → y=3；3 > 2 → 留在手上，繼續撞 w=2
w=2   撞 → y=1；1 ≤ 2 → 丟回 cnt[1]，重掃 w=2（cnt[2] 已剩 1）
w=2   手上空，cnt=1 → y=2
w=1   手上 y=2 > 1，撞 → y=1；1 ≤ 1 → 丟回 cnt[1]，重掃 w=1（cnt[1] 仍為 1）
w=1   手上空，cnt=1 → y=1
答案 1
```

```cpp
// Time: O(n + W)
// Space: O(W)
class Solution {
 public:
  int lastStoneWeight(vector<int>& stones) {
    array<int, 1001> cnt{};
    for (int s : stones) {
      ++cnt[s];
    }
    int y = 0;  // 手上的石頭（目前最重），0 表示沒有
    for (int w = 1000; w > 0;) {
      if (cnt[w] == 0) {
        --w;
        continue;
      }
      if (y == 0) {
        y = cnt[w] % 2 ? w : 0;  // 同重量兩兩抵消，剩單數就拿一顆在手上
        cnt[w] = 0;
      } else {
        --cnt[w];
        y -= w;       // 手上的 y 必大於 w，撞掉一顆 w
        if (y <= w) { // 殘餘不再是最重，丟回桶並重掃這格
          ++cnt[y];
          y = 0;
        }
      }
    }
    return y;
  }
};
```

> [!warning]
> 撞完的殘餘 `y − w` 只保證小於 `y`，**不保證小於 `w`**（例如 5 撞 2 剩 3）。若無條件丟回桶，它落在已經掃過的格子就再也取不到。所以殘餘若仍大於 `w` 就留在手上繼續撞同一格，只有 `≤ w` 時才丟回桶。

> [!note]
> 這題 n ≤ 30 遠小於 W = 1000，桶計數實際上比 heap 慢，純粹是知道有這條路；只有當 n ≫ W 時才划算。

## Related Problems

- [[0703-Kth-Largest-Element-in-a-Stream]] — 同樣用 heap 維護候選池，但那題固定容量、這題會縮小到空
- [[0215-Kth-Largest-Element-in-an-Array]] — heap 取最大的基本題，也有桶計數／快選的替代解
- [[1962-Remove-Stones-to-Minimize-the-Total]] — 同樣是反覆取堆頂、改值後推回堆的模式
- [[1049-Last-Stone-Weight-II]] — 同名但要求「最小」殘餘，變成 0/1 背包 DP，不能貪心用 heap
