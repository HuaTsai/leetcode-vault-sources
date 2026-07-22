---
leetcode-id: 206
difficulty: easy
tags:
  - linked-list
  - grind-169
  - neetcode-150
memo: 迭代版用 prev／head／next 三指針逐個掉頭，關鍵是改 next 前先把後繼存起來，否則鏈當場斷掉；遞迴版在回程用 head→next→next＝head 反接，且必須補上 head→next＝nullptr，不然原本的前兩節點互指成環
dg-publish: true
---

## Problem Description

Given the `head` of a singly linked list, reverse the list, and return the reversed list.

## Solution

核心觀念：反轉整條鏈其實只是**對每個節點做一次「把 next 指回前一個」**。單向鏈結串列沒有回頭路，所以在改寫 `head->next` 之前一定要先把後繼節點存起來，否則後半段立刻失聯。整個過程維護三個指標：`prev`（已反轉部分的頭）、`head`（正在處理的節點）、`next`（暫存的後繼）。

```txt
反轉 1 -> 2 -> 3，處理到節點 2 時的狀態：

       prev      head      next
        ↓         ↓         ↓
  nullptr <- 1    2    ->   3
           已反轉        尚未處理

每一輪固定做四件事：
  1. next = head->next    先存好後繼（否則下一步就找不到它了）
  2. head->next = prev    掉頭
  3. prev = head          prev 前進
  4. head = next          head 前進
```

> [!important] 先存後繼，再改指標
> 這是所有鏈結串列改指標題的通則。一旦 `head->next` 被覆蓋，原本的後半段就沒有任何指標指著它了。四個步驟的順序不能調換。

> [!tip] 迴圈結束時答案在 `prev` 不在 `head`
> 迴圈條件是 `while (head)`，跳出時 `head == nullptr`，此時最後一個被反轉的節點是 `prev`，也就是新的頭。回傳 `head` 會直接吃到空串列。

### 方法一：迭代三指針 — O(n)／O(1)（推薦）

一趟走完，只用常數額外空間，是本題的標準解。空串列（`head == nullptr`）自然不進迴圈、直接回傳 `prev = nullptr`，不需要特判。

```cpp
// Time: O(n)   每個節點恰好被處理一次
// Space: O(1)  只用三個指標
class Solution {
 public:
  ListNode* reverseList(ListNode* head) {
    ListNode *prev = nullptr;
    while (head) {
      auto next = head->next;
      head->next = prev;
      prev = head;
      head = next;
    }
    return prev;
  }
};
```

### 方法二：遞迴 — O(n)／O(n)

先一路遞迴到尾端拿到新的頭，**在回程時才動指標**。此時 `head->next` 是「已經反轉好的那段的尾巴」，所以 `head->next->next = head` 就把自己接到那條尾巴後面，再把 `head->next` 斷成 `nullptr`。`newHead` 一路原封不動往上傳。

```txt
reverseList(1 -> 2 -> 3) 的回程，處理到 head = 2：

  遞迴回來後：  1 -> 2 -> 3        （3 已是反轉好的段，newHead = 3）
                      ↑ head->next 就是那段的尾巴

  head->next->next = head：  3 -> 2      把自己接到尾巴後
  head->next = nullptr：     1    2 -> ..  切斷舊的正向指標
                                （否則 2 -> 3 且 3 -> 2，成環）
```

```cpp
// Time: O(n)   每個節點恰好回程一次
// Space: O(n)  遞迴堆疊深度等於串列長度
class Solution {
 public:
  ListNode* reverseList(ListNode* head) {
    if (!head || !head->next) return head;
    ListNode* newHead = reverseList(head->next);
    head->next->next = head;
    head->next = nullptr;
    return newHead;
  }
};
```

> [!warning] 漏掉 `head->next = nullptr` 會造成環狀鏈
> 只寫 `head->next->next = head` 的話，原本的 `head->next` 還指著後繼，兩個節點互相指向對方。回傳的 list 從新頭走下來到最後兩個節點就會無限繞圈，判題端讀取時直接掛掉或超時。

> [!note] 遞迴深度是實務上的隱憂
> 本題 `n ≤ 5000`，遞迴不會爆堆疊（實測 5000 節點正常）。但同樣寫法套到百萬級的鏈上就會 stack overflow，這是迭代版除了 O(1) 空間之外更該被當成首選的理由。

## Related Problems

[[0092-Reverse-Linked-List-II]] — 只反轉 `[left, right]` 區間，本題的一般化，需要多接回前後兩端
[[0234-Palindrome-Linked-List]] — 找中點後反轉後半段再逐一比對，直接把本題當子程序用
[[0143-Reorder-List]] — 找中點＋反轉後半＋交錯合併，本題是三步驟中的一步
[[0025-Reverse-Nodes-in-k-Group]] — 每 k 個節點反轉一次，本題的分段強化版
[[0021-Merge-Two-Sorted-Lists]] — 同為「小心搬指標」的鏈結串列基本功
