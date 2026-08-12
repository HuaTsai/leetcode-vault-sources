---
leetcode-id: 21
difficulty: easy
tags:
  - linked-list
  - grind-169
  - neetcode-150
memo: 用 dummy node 當前哨免掉 head 的特判，全程只重接指標不配置新節點；相等時取 list1 才是 stable merge，尾端用三元運算子無條件接上剩餘那串，比 if／else if 穩健（兩串皆空時 dummy 的 next 才不會從頭到尾沒被賦值）
dg-publish: true
---

## Problem Description

You are given the heads of two sorted linked lists `list1` and `list2`.

Merge the two lists into one sorted list. The list should be made by splicing together the nodes of the first two lists.

Return the head of the merged linked list.

## Solution

核心觀念：兩條各自有序的鏈，合併時**答案的下一個節點必定是兩個 head 中較小的那個**——因為各自已排序，其餘節點都不會比自己的 head 更小。所以只要反覆比較兩個 head、把較小的接到結果尾端、該串前進一格即可。題目說 splicing together，代表**不配置新節點、只重接 `next` 指標**，所以額外空間是 O(1)。

麻煩的只有「結果的第一個節點是誰」要看比較結果，接頭時得特判。**dummy node（前哨節點）** 就是為了消掉這個特判：先造一個不屬於答案的假頭，所有接線都對「`cur->next`」做，最後回傳 `dummy.next`，第一個節點與後面的節點走完全相同的程式路徑。

```txt
list1: 1 → 2 → 4
list2: 1 → 3 → 4

dummy → ?          每輪取兩個 head 中較小者接到 cur 之後，該串前進一格
 ↑cur

  l1=1, l2=1   相等取 list1  → dummy → 1(l1)
  l1=2, l2=1   取 list2      →     ⋯ → 1(l2)
  l1=2, l2=3   取 list1      →     ⋯ → 2(l1)
  l1=4, l2=3   取 list2      →     ⋯ → 3(l2)
  l1=4, l2=4   相等取 list1  →     ⋯ → 4(l1)
  l1 已空      剩下整串接上  →     ⋯ → 4(l2)

回傳 dummy.next（不是 &dummy）
```

### 方法一：Dummy Node 迭代 — O(n+m)／O(1)（推薦）

一趟走完，只用兩個指標。迴圈跑到任一串耗盡就結束，剩下那串**整條直接接上**——它本來就有序，不必逐個搬。

```cpp
// Time: O(n + m)  兩串各自的節點最多各被走過一次
// Space: O(1)     只重接指標，沒有配置新節點
class Solution {
 public:
  ListNode* mergeTwoLists(ListNode* list1, ListNode* list2) {
    ListNode dummy;
    ListNode* cur = &dummy;
    while (list1 && list2) {
      if (list1->val <= list2->val) {  // 相等取 list1
        cur->next = list1;
        list1 = list1->next;
      } else {
        cur->next = list2;
        list2 = list2->next;
      }
      cur = cur->next;
    }
    cur->next = list1 ? list1 : list2;  // 至少一邊已是 nullptr
    return dummy.next;
  }
};
```

> [!important] 相等時取 `list1` 才是 stable merge
> 寫 `<` 而非 `<=`，值相等時會改取 `list2` 的節點。本題只比 `val`，兩者輸出的數列一模一樣照樣 AC，但**節點順序不同**：`[1,3]` 與 `[1,2]` 合併後，`<` 版的第一個節點是 list2 的 1。一旦節點帶有 `val` 以外的資料（或這段被拿去當 merge sort 的合併步驟），相等時保留左側先來的順序就是**穩定性**的來源。用 `<=` 是零成本的好習慣。

> [!tip] 尾端用三元運算子無條件賦值，別用 `if / else if`
> 寫成 `if (list1) cur->next = list1; else if (list2) cur->next = list2;` 時，**兩串皆空**的情況下兩個分支都不成立，`dummy.next` 從頭到尾沒被寫過，回傳值就完全依賴 `ListNode` 的預設建構子。LeetCode 的定義是 `ListNode() : val(0), next(nullptr) {}` 所以安全，但這是對外部定義的隱性依賴。`cur->next = list1 ? list1 : list2;` 無條件寫入，接上 `nullptr` 本來就是正確結果，一行同時處理三種收尾情況。

> [!note] `dummy` 是 stack 上的物件，不是 `new` 出來的
> 寫 `ListNode dummy;` 就好，不需要 `new ListNode()`——後者要記得 `delete`，忘了就洩漏。回傳的是 `dummy.next`（真正的第一個節點），`&dummy` 本身在函式結束後就失效，**絕不能回傳**。

### 方法二：遞迴 — O(n+m)／O(n+m)

換個角度看：合併結果的頭是兩個 head 中較小者，而它的 `next` 就是「剩下部分的合併結果」——遞迴定義天生成立。空串列是自然的 base case，連 dummy node 都省了。

```cpp
// Time: O(n + m)      每個節點恰好被決定一次
// Space: O(n + m)     遞迴堆疊深度等於總節點數
class Solution {
 public:
  ListNode* mergeTwoLists(ListNode* list1, ListNode* list2) {
    if (!list1) return list2;
    if (!list2) return list1;
    if (list1->val <= list2->val) {
      list1->next = mergeTwoLists(list1->next, list2);
      return list1;
    }
    list2->next = mergeTwoLists(list1, list2->next);
    return list2;
  }
};
```

> [!warning] 遞迴深度等於總節點數，不是對數
> 本題 `n, m ≤ 50` 完全沒問題，但這個遞迴**不是尾遞迴的自然形式**、深度隨長度線性成長，套到長串列上會 stack overflow。寫法漂亮，實務上仍以方法一為首選。

## Related Problems

- [[0023-Merge-k-Sorted-Lists]] — 從兩串推廣到 k 串，用 min-heap 或兩兩分治合併，內層合併就是本題
- [[0148-Sort-List]] — 鏈結串列版 merge sort，切半後的合併步驟直接套用本題
- [[0088-Merge-Sorted-Array]] — 陣列版的同一件事，因為要原地寫入 nums1，改成從尾端往前填才不會覆蓋
- [[0206-Reverse-Linked-List]] — 同為「小心搬指標」的鏈結串列基本功
