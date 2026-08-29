---
leetcode-id: 23
difficulty: hard
tags:
  - linked-list
  - heap
  - merge-sort
  - tournament-sort
  - grind-169
  - neetcode-150
memo: k 路合併的核心是堆裡只放「每條串的當前 head」，大小恆 ≤ k，所以是 O(N log k) 而不是 O(N log N)；min-heap 的比較器要寫 `a->val > b->val`（反直覺），而且 priority_queue 是 multiset 語意、重複值一個都不會少；迴圈能自然收尾——最後被 pop 的節點必定是某條串的尾端，next 本來就是 nullptr，不必手動補
dg-publish: true
---

## Problem Description

You are given an array of `k` linked-lists `lists`, each linked-list is sorted in ascending order.

Merge all the linked-lists into one sorted linked-list and return it.

Constraints:

- lists[i] is sorted in ascending order.

## Solution

核心觀念是 [[0021-Merge-Two-Sorted-Lists]] 的直接推廣：**答案的下一個節點，必定是 k 條串「當前 head」中最小的那個**——每條串各自有序，一條串裡排在 head 後面的節點都不可能比自己的 head 更小，所以只有這 k 個是候選。

於是問題退化成一個很單純的資料結構問題：**維護 k 個候選值，反覆取最小、然後補進一個新值**。這正是 min-heap 的定義（方法一）。另一條路是換個切法——既然兩兩合併已經會寫，就想辦法讓每個節點被搬的次數變少，用分治把合併排成一棵樹（方法二）。兩者都是 O(N log k)，`N` 是總節點數。

```txt
lists = [ 1ₐ→ 4ₐ→ 5ₐ ,  1ᵦ→ 3ᵦ→ 4ᵦ ,  2ᵧ→ 6ᵧ ]        （下標標示節點來自哪一條串）

每一步只在「k 個當前 head」裡挑最小 —— 堆的大小恆 ≤ k，與 N 無關

   堆內容        取出          答案尾端            補回同一條串的下一個
  ──────────────────────────────────────────────────────────────────
   {1ₐ,1ᵦ,2ᵧ}     1ₐ    →     1ₐ                    push 4ₐ
   {1ᵦ,2ᵧ,4ₐ}     1ᵦ    →     1ₐ→1ᵦ                 push 3ᵦ
   {2ᵧ,3ᵦ,4ₐ}     2ᵧ    →     1ₐ→1ᵦ→2ᵧ              push 6ᵧ
   {3ᵦ,4ₐ,6ᵧ}     3ᵦ    →     1ₐ→1ᵦ→2ᵧ→3ᵦ           push 4ᵦ
   {4ₐ,4ᵦ,6ᵧ}     4ₐ    →     ⋯→4ₐ                  push 5ₐ
   {4ᵦ,5ₐ,6ᵧ}     4ᵦ    →     ⋯→4ₐ→4ᵦ               β 串已盡，不補
   {5ₐ,6ᵧ}        5ₐ    →     ⋯→5ₐ                  α 串已盡，不補
   {6ᵧ}           6ᵧ    →     ⋯→6ᵧ                  γ 串已盡，堆空 → 結束
```

### 方法一：min-heap（k 路合併）— O(N log k)／O(k)

把 k 條串的 head 全部丟進 min-heap，每次 pop 最小的接到答案尾端，再把「它的下一個節點」push 回去補位。堆裡永遠是「每條還沒走完的串各出一個代表」。

```cpp
// Time: O(N log k)  N 個節點各 push/pop 一次，堆高恆為 log k
// Space: O(k)       堆裡每條串最多佔一格；答案只重接指標，不配置節點
class Solution {
 public:
  ListNode* mergeKLists(vector<ListNode*>& lists) {
    auto cmp = [](const ListNode *a, const ListNode *b) {
      return a->val > b->val;  // min-heap
    };
    priority_queue<ListNode *, vector<ListNode *>, decltype(cmp)> pq(cmp);

    for (const auto &l : lists) {
      if (l) {
        pq.push(l);
      }
    }

    ListNode dummy(0);
    ListNode *cur = &dummy;
    while (!pq.empty()) {
      auto node = pq.top();
      pq.pop();
      if (node->next) {
        pq.push(node->next);
      }
      cur->next = node;
      cur = node;
    }
    return dummy.next;
  }
};
```

> [!important] 為什麼是 log k 而不是 log N
> 很容易誤以為「把所有節點丟進堆裡排序」，那是 O(N log N)。這裡堆的內容始終是**每條串各一個 head**，pop 一個才 push 一個，所以大小上界是 k 而非 N。這個「候選集只保留每路一個代表」的骨架，是所有 k 路合併問題的共通形狀。

> [!warning] `>` 給出的是 min-heap，方向與直覺相反
> `priority_queue` 的 `Compare` 定義的是「誰**優先度低**」——`cmp(a, b)` 為 true 代表 a 排在 b 後面。所以想要小的先出來，比較器要寫 `a->val > b->val`。寫成 `<` 會得到 max-heap，輸出變降序。註解裡的 `// min-heap` 標得好，這行是最容易寫反、又最難從輸出一眼看出錯在哪的地方。
> 另外比較器必須是**嚴格弱序**：寫成 `>=` 時，heap 演算法的索引不會越界、通常安靜地跑完（同一份 `>=` 丟進 `sort` 則會被 `_GLIBCXX_DEBUG` 當場抓出 irreflexive 違規），但標準上已是 UB。沒炸不等於合法，見 [[STL-Pitfalls]]。

> [!tip] 迴圈自然收尾，不必手動補 `cur->next = nullptr`
> 中途接上的節點，`next` 都還指著它在原串裡的後繼（那個節點正躺在堆裡），看起來是「錯的」——但它下一輪被 pop 出來時就會被重新覆寫，所以只是暫態。真正的關鍵在最後一輪：**迴圈能結束，代表最後 pop 出的節點沒有 `next` 可 push**，也就是它本來就是某條串的尾端、`next` 已經是 `nullptr`。所以尾端天生正確，多寫一行收尾是冗餘的。

> [!note] `priority_queue` 是 multiset 語意，重複值一個都不會少
> 它是 container adaptor，`push` 就是 `push_back` + `push_heap`，比較器只回答「誰先出來」，從不做等價判定（那是 `set` / `map` 才有的 `!cmp(a,b) && !cmp(b,a)` 去重）。所以 `[[1,1],[1],[1]]` 這種輸入，四個 `val == 1` 的節點會同時待在堆裡、四個都被接進答案。反過來說：**平手時的相對順序不保證**（heap 不是 stable 的），本題無所謂，但若要求「值相同時保留原輸入順序」，就得把 `pair<val, list_index>` 一起比。

> [!tip] 兩個可選的打磨
> 1. **C++20 起可以省略 `pq(cmp)`**：無捕獲 lambda 從 C++20 開始是 default-constructible，寫 `priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq;` 即可（C++17 編同一行會直接 no matching function 編譯失敗）。
> 2. **一次 heapify 取代 k 次 push**：先把非空 head 收進 `vector`，再用 `priority_queue pq(cmp, std::move(heads));`，建堆從 O(k log k) 降到 O(k)。相對於主迴圈的 O(N log k) 只是常數項，實測看不出差別，知道有這個建構子就好。
>
> `ListNode dummy(0);` 用 stack 上的物件、明確給初值，比 `new ListNode()`（要記得 delete）和 `ListNode dummy;`（依賴外部定義的預設建構子）都穩健，這裡寫得比一般範例還謹慎。

### 方法二：分治兩兩合併 — O(N log k)／O(1)

不用堆也能到同樣的複雜度：反覆把 k 條串**兩兩配對**合併，每輪串數減半，log k 輪之後只剩一條。內層的合併子程序就是 [[0021-Merge-Two-Sorted-Lists]] 原封不動。

```txt
循序合併（O(kN)，錯誤示範）         分治合併（O(N log k)）

acc  = L1                          第 1 輪   L1 L2 L3 L4 L5 L6 L7 L8    8 條
acc += L2   → 掃過 2L                        └──┴──┴──┴──╯ 前半配後半    ← 全體節點被搬 1 次
acc += L3   → 掃過 3L              第 2 輪   M1 M2 M3 M4                 4 條
acc += L4   → 掃過 4L                        └──┴──╯                     ← 全體節點被搬 1 次
    ⋯                              第 3 輪   M1 M2                       2 條
acc += Lk   → 掃過 kL                        └──╯                        ← 全體節點被搬 1 次
                                   結果      M1                          1 條
總搬運 L(2+3+⋯+k) = O(kN)          共 log k 輪，總搬運 O(N log k)
```

```cpp
// Time: O(N log k)  共 log k 輪，每輪所有節點合計被搬過一次
// Space: O(1)       就地覆寫 lists，迭代版連遞迴堆疊都不用
class Solution {
 public:
  ListNode* mergeKLists(vector<ListNode*>& lists) {
    if (lists.empty()) return nullptr;
    int n = lists.size();
    while (n > 1) {
      int m = (n + 1) / 2;  // 本輪結束後剩下的串數（奇數時最中間那條輪空）
      for (int i = 0; i < n / 2; ++i) {
        lists[i] = merge2(lists[i], lists[i + m]);  // 頭尾配對，輪空的留在原位
      }
      n = m;
    }
    return lists[0];
  }

 private:
  ListNode* merge2(ListNode *a, ListNode *b) {
    ListNode dummy(0);
    ListNode *cur = &dummy;
    while (a && b) {
      if (a->val <= b->val) {
        cur->next = a;
        a = a->next;
      } else {
        cur->next = b;
        b = b->next;
      }
      cur = cur->next;
    }
    cur->next = a ? a : b;
    return dummy.next;
  }
};
```

> [!warning] 循序合併（`acc = merge2(acc, lists[i])` 跑一圈）是這題最經典的 TLE 陷阱
> 看起來只是「把 0021 呼叫 k 次」，但第 i 次合併時累積串已經有 i·L 個節點，全部要再掃一遍——總搬運量是 L(2+3+⋯+k) = **O(kN)**。分治的差別只在「配對的順序」，卻讓每個節點從被搬 O(k) 次降到 O(log k) 次。實測 k=1000、N=100000 時，循序 352 ms、分治 7.9 ms，差 45 倍。

> [!note] 兩種解法實際上跑得一樣快，不必迷信哪個「比較快」
> 常見說法是「分治比 heap 快」，實測（`-O2`，N 固定 100k，每組取 7 次中位數）並非如此：
>
> | k | 每串長度 | min-heap | 分治 | 循序 |
> |---|---|---|---|---|
> | 10 | 10000 | 2.01 ms | 1.70 ms | 2.01 ms |
> | 100 | 1000 | 4.24 ms | 4.31 ms | 23.5 ms |
> | 1000 | 100 | 6.75 ms | 7.86 ms | 352 ms |
> | 10000 | 10 | 10.3 ms | 10.5 ms | — |
>
> 兩者在同一個數量級內互有勝負：k 小時分治略快（合併是純線性掃描、沒有 heap 的 sift 成本），k 大時 heap 追平甚至反超。倒是**節點的記憶體佈局**影響更明顯——同樣 k=1000，節點連續配置時分治 5.4 ms／heap 7.1 ms，交錯配置（模擬各自 `new` 的真實情況）反過來變成分治 8.0 ms／heap 6.8 ms。分治靠的是循序掃描的區域性，一旦節點散落就吃虧。
>
> 結論是**照自己順手的寫**：heap 版更通用（可推廣到串流、外部排序、k 路以外的候選集），分治版空間 O(1) 且不依賴 comparator 寫對方向。詳見 [[Benchmarking-Toolkit]]。

## Related Problems

- [[0021-Merge-Two-Sorted-Lists]] — 本題 k = 2 的特例，方法二的 `merge2` 就是它原封不動
- [[0148-Sort-List]] — 鏈結串列版 merge sort，同樣是分治＋兩兩合併，只是這題已經幫你切好了
- [[0703-Kth-Largest-Element-in-a-Stream]] — heap 的基本操作練習，同樣靠「維護固定大小的候選集」
- [[0347-Top-K-Frequent-Elements]] — heap 的另一種用法，維護大小為 k 的堆做 top-k 而非 k 路合併
- [[0378-Kth-Smallest-Element-in-a-Sorted-Matrix]] — 每列當一條有序串，完全相同的「k 個指標 + min-heap」骨架
- [[0632-Smallest-Range-Covering-Elements-from-K-Lists]] — 同骨架再加一個視窗最大值，是這題的直接進階版
- [[STL-Pitfalls]] — comparator 的嚴格弱序要求、`priority_queue` 的比較方向
