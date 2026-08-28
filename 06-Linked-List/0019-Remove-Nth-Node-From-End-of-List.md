---
leetcode-id: 19
difficulty: medium
tags:
  - linked-list
  - two-pointers
  - grind-169
  - neetcode-150
memo: 單向鏈結串列無法回頭，所以用「兩個指標拉開固定 n 步」把倒數第 n 變成同步走訪；dummy 節點讓刪掉 head 不必特判，兩指標都從 dummy 出發、lead 先走 n 步，走到 lead 是尾節點時 trail 正好停在待刪節點的前驅——領先步數與停止條件是配成一組的，多走一步就刪錯一位；注意這裡是同速定距，和 0141 的差速龜兔是不同機制
dg-publish: true
---

## Problem Description

Given the `head` of a linked list, remove the `nth` node from the end of the list and return its head.

## Solution

核心觀念：單向串列只能往前走，「倒數第 n 個」沒辦法直接定位。解法是**把倒數轉成正數**——讓兩個指標拉開固定 n 步的間距後同速前進，**前面那個走到底時，後面那個就落在倒數第 n 附近**。間距在整趟過程中不變，所以只需一趟。

另一半的關鍵是：刪除單向串列的節點，手上必須是它的**前驅**。所以間距要調到讓慢指標停在前驅，而不是停在待刪節點本身。

```txt
dummy -> 1 -> 2 -> 3 -> 4 -> 5 -> null      n = 2

lead 先從 dummy 走 2 步：
 trail       lead
   |           |
 dummy -> 1 -> 2 -> 3 -> 4 -> 5             間距固定為 2

同速走到 lead->next == nullptr（lead 踩在尾節點）：
                  trail     lead
                    |         |
 dummy -> 1 -> 2 -> 3 -> 4 -> 5
                         ^ 待刪的 4（倒數第 2）就是 trail->next
```

### 方法一：dummy ＋ 定距雙指標，一趟 — O(n)／O(1)（推薦）

```cpp
// Time: O(n)   lead 走完整條串列，trail 走 len - n 步，合計一趟
// Space: O(1)  只有兩個指標與一個堆疊上的 dummy
class Solution {
 public:
  ListNode* removeNthFromEnd(ListNode* head, int n) {
    ListNode dummy(0, head);
    ListNode *trail = &dummy;
    ListNode *lead = &dummy;
    while (n--) {
      lead = lead->next;
    }
    while (lead->next) {
      trail = trail->next;
      lead = lead->next;
    }
    auto node = trail->next;
    trail->next = node->next;
    delete node;
    return dummy.next;
  }
};
```

> [!important] dummy 的價值就是讓「刪掉 head」退化成普通情況
> 待刪節點是 head 時（`n == 串列長度`），它沒有前驅，`prev->next = ...` 這套邏輯就無處著力。把 `trail` 從 `&dummy` 起跑，等於**替 head 憑空造了一個前驅**，於是刪 head、刪中間、刪尾巴走的是同一行程式碼。不用 dummy 也能寫，但一定得補上一行特判：
> ```cpp
> ListNode *fast = head, *slow = head;
> while (n--) fast = fast->next;
> if (!fast) return head->next;   // <- 少了這行，下一行就對 nullptr 解參考
> while (fast->next) { slow = slow->next; fast = fast->next; }
> slow->next = slow->next->next;
> return head;
> ```
> 兩種寫法都驗證過，行為完全一致。差別只在**特判是寫成一行 `if`，還是寫成一個 dummy 節點**——後者不用去想「什麼時候會沒有前驅」，這正是 [[0021-Merge-Two-Sorted-Lists]]、[[0203-Remove-Linked-List-Elements]] 也在用同一招的原因。

> [!warning] 領先幾步和停止條件是一組的，拆開來記必定 off-by-one
> 這題最常見的錯不是想不到雙指標，而是**間距和迴圈條件配錯**。兩組正確的組合：
> ```txt
>  起點      lead 先走   停止條件             此時 trail 在
>  -------------------------------------------------------------
>  dummy     n 步        lead->next == null   倒數第 n 的前驅   <- 本篇寫法
>  dummy     n + 1 步    lead == null         倒數第 n 的前驅   <- 等價變體
> ```
> 兩者都驗證過（隨機 7915 組 `len` 與 `n` 的組合全數一致）。但若領先 `n` 步卻寫成 `while (lead)`，trail 會多走一步：`[1,2,3,4,5]` 刪 `n = 3` 得到 `[1,2,3,5]`（刪掉了倒數第 2），而 `n = 1` 時 `trail` 直接走到尾節點、`trail->next` 是 nullptr，下一行立刻爆炸。**記法：讓 lead 停在「最後一個節點」還是「null」，決定它該先走 n 步還是 n + 1 步。**

> [!tip] 命名成 `lead` / `trail` 而不是 `tort` / `hare`，因為這裡不是龜兔賽跑
> Floyd 的龜兔（[[0141-Linked-List-Cycle]]、[[0876-Middle-of-the-Linked-List]]）靠的是**速度差**：一次一步 vs 一次兩步，間距會持續拉大，用來偵測環或找中點。本題兩個指標**速度完全相同**，只是起跑就錯開 n 步，間距全程不變。同樣掛著「雙指標」的名字，機制卻是兩回事：**差速用來讓兩者相遇，定距用來鎖住一個相對位置。** 沿用 `slow` / `fast` 這組名字會把兩類題在腦中混成一團，`lead` / `trail`（或 `ahead` / `behind`）才說得出這裡真正發生的事——一個在前探路、一個在後跟著，距離不變。

> [!note] 先存指標再改接線，`delete` 一定放最後
> `auto node = trail->next;` 這一行不只是為了 `delete`——存下來之後，改接線就能寫成 `trail->next = node->next;`，不必再走一次 `trail->next->next`。三行的順序是有意義的：**存指標 → 改接線 → 釋放**。`delete` 之後 `node` 就是懸空指標，還去讀 `node->next` 就是 UB。至於該不該 `delete`：面試時主動釋放是加分項，LeetCode 的 judge 只從你回傳的 head 走訪輸出，刪掉的節點不會再被碰到。

### 方法二：兩趟，先數長度再走 `len - n` 步 — O(n)／O(1)

把「倒數第 n」老實換算成「正數第 `len - n + 1`」，前驅就是從 dummy 走 `len - n` 步。複雜度和方法一完全相同（兩趟仍是 O(n)），只是多讀一次串列。

```cpp
// Time: O(n)   一趟數長度、一趟走到前驅
// Space: O(1)
class Solution {
 public:
  ListNode* removeNthFromEnd(ListNode* head, int n) {
    int len = 0;
    for (ListNode *p = head; p; p = p->next) {
      ++len;
    }
    ListNode dummy(0, head);
    ListNode *prev = &dummy;
    for (int i = 0; i < len - n; ++i) {
      prev = prev->next;
    }
    ListNode *victim = prev->next;
    prev->next = victim->next;
    delete victim;
    return dummy.next;
  }
};
```

> [!tip] 面試被要求 one pass 時，方法一才是答案；但先寫方法二不丟臉
> 題目的進階要求是 "Could you do this in one pass?"，所以方法一是被期待的解。不過方法二的價值在於**它幾乎不可能寫錯**：`len - n` 這個換算是明擺著的算術，而方法一的間距要靠腦內模擬。實務上兩者常數差異極小（都是走 O(n) 個指標）。卡住時先用方法二把正確性拿下、再改成一趟，比對著空氣數步數強。

> [!note] 還有一個遞迴寫法，代價是 O(n) 堆疊
> 遞迴的**回歸階段天生就是由後往前**，所以讓每層回傳「自己是倒數第幾個」就能直接比對 `n`：
> ```cpp
> int rec(ListNode *node, int n, ListNode *prev) {
>   if (!node) { return 0; }
>   int idx = rec(node->next, n, node) + 1;
>   if (idx == n) { prev->next = node->next; delete node; }
>   return idx;
> }
> ```
> 外層一樣用 dummy 當 `prev` 起點。想法漂亮且驗證無誤，但把 O(1) 空間換成 O(n) 遞迴堆疊，串列一長就有爆堆疊風險，正式解不採用。

## Related Problems

- [[0141-Linked-List-Cycle]] — 差速龜兔的原型題，正好是本題「同速定距」的對照組
- [[0876-Middle-of-the-Linked-List]] — 另一個差速應用，一次兩步的快指標到底時慢指標剛好在中間
- [[0021-Merge-Two-Sorted-Lists]] — dummy 節點的另一個經典用途，省掉「第一個節點怎麼接」的特判
- [[0203-Remove-Linked-List-Elements]] — 同樣用 dummy 把「刪掉 head」統一進一般情況，且可能連刪多個
- [[0061-Rotate-List]] — 一樣要定位倒數第 k 個，先數長度再對 k 取模，接法比本題更繞
