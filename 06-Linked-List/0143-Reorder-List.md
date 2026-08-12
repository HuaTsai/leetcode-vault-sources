---
leetcode-id: 143
difficulty: medium
tags:
  - linked-list
  - grind-169
  - neetcode-150
memo: 三個經典子題的組合，快慢指針找中點（停在左半最後一個才切得斷）／反轉後半／左先右後交錯；左半長度永遠不小於右半，所以迴圈只看右半有沒有用完，交錯時順手把右節點接到下一個左節點就免掉收尾特判，切斷 slow 的 next 忘了會成環
dg-publish: true
---

## Problem Description

You are given the head of a singly linked-list. The list can be represented as:

L0 → L1 → … → Ln - 1 → Ln
Reorder the list to be on the following form:

L0 → Ln → L1 → Ln - 1 → L2 → Ln - 2 → …
You may not modify the values in the list's nodes. Only nodes themselves may be changed.

## Solution

核心觀念：目標排列是「頭一個、尾一個、頭二個、尾二個…」交替。單向鏈結串列沒有回頭路，**要一路往回取 Ln、Ln-1 只能先把後半段反轉**。把需求拆開來看就是三個各自獨立的經典子題：

1. **找中點**把串列切成左右兩半（快慢指針）
2. **反轉右半**，讓它變成從尾巴往前走
3. **左先右後交錯**接回去

三步都只重接指標、不配置新節點，所以全程 O(n) 時間、O(1) 空間。

```txt
原始：  1 → 2 → 3 → 4 → 5
目標：  1 → 5 → 2 → 4 → 3

① 找中點（slow 停在左半的最後一個節點）
        1 → 2 → 3 → 4 → 5
                ↑slow       ↑fast
    切斷 slow->next 後：  左 1 → 2 → 3    右 4 → 5

② 反轉右半
    左 1 → 2 → 3        右 5 → 4

③ 交錯（左先右後，左半用完為止）
    1 → 5 → 2 → 4 → 3
```

### 方法一：找中點 → 反轉後半 → 交錯 — O(n)／O(1)（推薦）

```cpp
// Time: O(n)   找中點、反轉、交錯各掃一趟
// Space: O(1)  只重接指標
class Solution {
 public:
  void reorderList(ListNode* head) {
    if (!head || !head->next) return;

    // ① 找中點：slow 停在左半的最後一個節點
    ListNode* slow = head;
    ListNode* fast = head;
    while (fast->next && fast->next->next) {
      slow = slow->next;
      fast = fast->next->next;
    }

    // ② 反轉右半後切斷左右
    ListNode* second = reverseList(slow->next);
    slow->next = nullptr;

    // ③ 交錯：接右節點的同時，順手把它接到下一個左節點
    ListNode* first = head;
    while (second) {
      ListNode* n1 = first->next;
      ListNode* n2 = second->next;
      first->next = second;
      second->next = n1;
      first = n1;
      second = n2;
    }
  }

 private:
  ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr;
    while (head) {
      ListNode* next = head->next;
      head->next = prev;
      prev = head;
      head = next;
    }
    return prev;
  }
};
```

> [!important] 左半長度永遠不小於右半，這是收尾不必特判的原因
> `while (fast->next && fast->next->next)` 的停位讓左半在 n 為奇數時多一個節點（n=5 切成 `1,2,3` 與 `4,5`）。因此：交錯時「左先右後」永遠合法；迴圈條件只需 `while (second)`，因為 `second` 還在就保證 `first` 也還在；最後多出來的那個左節點，它的 `next` 在切斷時就已是 `nullptr`，而且**在上一輪的 `second->next = n1` 已經被接上了**，不需要任何收尾補接。

> [!tip] 快慢指針的停位決定切不切得斷
> 用 `while (fast->next && fast->next->next)`，`slow` 停在**左半的最後一個**，`slow->next` 就是右半的頭、`slow->next = nullptr` 就是切斷，一個指標搞定。若改成更常見的 `while (fast && fast->next)`，`slow` 會停在**右半的第一個**，此時手上沒有它的前驅，得額外維護一個 `prev` 才切得斷。挑哪一種取決於你要的是「左半末端」還是「右半開頭」。

> [!warning] 忘記 `slow->next = nullptr` 會做出自環
> 不切斷的話，左半最後一個節點仍指著原本的後繼。以 `1→2→3→4` 為例，反轉後 `2` 還指著 `3`，交錯到最後一輪會取到 `n1 = 3` 並執行 `3->next = 3`，判題端一讀就無限繞圈。切斷這一步不是為了美觀，是正確性的一部分。

> [!note] 用 dummy node 的變體
> 也可以配一個 dummy、每輪接「上一輪的右 → 本輪左」與「左 → 右」兩條邊，`cur` 跟著跳到右節點。這種接法把「右 → 下一個左」留到下一輪才接，所以最後一個左節點沒人接，迴圈結束後要補一句 `if (first) cur->next = first;` 收尾。兩種都對，差別只在**每輪接哪兩條邊**——本文的寫法提前接掉，就省了 dummy 與收尾判斷。

### 方法二：vector 存指標雙向夾 — O(n)／O(n)

先把所有節點指標倒進 `vector`，就能隨機存取，用左右兩個索引往中間夾、交替接線。省掉反轉與快慢指針，思路最直白，代價是 O(n) 額外空間。

```cpp
// Time: O(n)
// Space: O(n)  額外存 n 個節點指標
class Solution {
 public:
  void reorderList(ListNode* head) {
    vector<ListNode*> v;
    for (ListNode* p = head; p; p = p->next) {
      v.push_back(p);
    }
    int l = 0, r = (int)v.size() - 1;
    while (l < r) {
      v[l]->next = v[r];
      ++l;
      if (l == r) break;  // 左右夾到同一個節點，該節點就是新的尾巴
      v[r]->next = v[l];
      --r;
    }
    if (!v.empty()) {
      v[l]->next = nullptr;  // 尾端一定要斷，否則成環
    }
  }
};
```

> [!warning] 兩種寫法都必須親手把新的尾巴設成 `nullptr`
> 重排後原本的尾節點不再是尾巴，若沒有明確切斷最後一個節點的 `next`，就會殘留舊指標而成環。方法一靠 `slow->next = nullptr` 完成，方法二靠最後那句 `v[l]->next = nullptr`。

## Related Problems

- [[0876-Middle-of-the-Linked-List]] — 第一步的獨立版，正好用來練兩種快慢指針停位的差別
- [[0206-Reverse-Linked-List]] — 第二步直接當子程序呼叫，本題是它最典型的應用場景
- [[0021-Merge-Two-Sorted-Lists]] — 第三步同款的交錯接線，只是挑哪一邊改成比大小
- [[0234-Palindrome-Linked-List]] — 一樣是找中點＋反轉後半，第三步換成逐一比對
- [[0025-Reverse-Nodes-in-k-Group]] — 分段反轉，指標搬移的強化版
