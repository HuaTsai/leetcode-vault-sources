---
leetcode-id: 141
difficulty: easy
tags:
  - linked-list
  - two-pointers
  - hash
  - grind-169
  - neetcode-150
memo: 快慢指針判環，fast 每步比 slow 多走一格，環上間距只會逐 1 遞減、必定歸零而不會跳過；比較一定要放在移動之後，否則兩者同起點第一輪就誤判 true，hash set 解法則用 insert 的回傳值一次查完在不在
dg-publish: true
---

## Problem Description

Given `head`, the head of a linked list, determine if the linked list has a cycle in it.

There is a cycle in a linked list if there is some node in the list that can be reached again by continuously following the `next` pointer. Internally, `pos` is used to denote the index of the node that tail's `next` pointer is connected to. Note that `pos` is not passed as a parameter.

Return `true` if there is a cycle in the linked list. Otherwise, return `false`.

Constraints:

- pos is `-1` or a valid index in the linked-list.

## Solution

核心觀念：單向鏈結串列只能往前走，「有環」等價於「往前走永遠走不到 `nullptr`」。最直覺的判法是把走過的節點記下來，再遇到就是環——正確但要 O(n) 空間。**Floyd 判圈（龜兔賽跑）**把空間壓到 O(1)：一個指標一次走一步、一個一次走兩步，無環時快的先撞到 `nullptr`，有環時快的會在環裡追上慢的。

```txt
無環：  1 → 2 → 3 → 4 → nullptr
        fast 先走到底 → false

有環：  3 → 2 → 0 → -4
            ↑         │
            └─────────┘
        兩人都進環後，fast 每步比 slow 多前進 1 格，
        環上間距 d 依 d → d-1 → … → 0 遞減 → 必定相遇 → true
```

### 方法一：Floyd 快慢指針 — O(n)／O(1)（推薦）

```cpp
// Time: O(n)   無環時 fast 走到底；有環時進環後最多再繞一圈就相遇
// Space: O(1)  只用兩個指標
class Solution {
 public:
  bool hasCycle(ListNode *head) {
    ListNode *slow = head;
    ListNode *fast = head;
    while (fast && fast->next) {
      slow = slow->next;
      fast = fast->next->next;
      if (slow == fast) {
        return true;
      }
    }
    return false;
  }
};
```

> [!important] fast 不可能跳過 slow，所以「相遇」是必然而非碰運氣
> 這是 Floyd 唯一需要證明的地方。兩者都進環之後，設 slow 領先 fast 的環上距離為 `d`（`0 ≤ d < 環長`）。每走一輪 slow 前進 1、fast 前進 2，**間距剛好縮 1**，於是 `d → d-1 → … → 0`，中途不可能一步跨過去，最多 `d` 輪必定相遇。也因此有環時的總步數是 O(n)，不會退化。
> 反過來看就知道步長不能亂改：若 fast 一次走 3 步，間距每輪縮 2、奇偶性不變，環長為偶數而間距為奇數時就永遠碰不到 0。

> [!warning] 比較必須放在移動之後
> `slow` 與 `fast` 都從 `head` 出發，若把 `if (slow == fast)` 挪到迴圈開頭（移動之前），第一輪就會拿 `head == head` 直接回傳 `true`，任何串列都判成有環。要嘛像上面「先動再比」，要嘛改成 `fast = head->next` 起手才能「先比再動」。

> [!tip] 迴圈條件不必檢查 slow
> `while (fast && fast->next)` 已經隱含 `slow` 非空——`slow` 走過的是 `fast` 路徑的前綴，`fast` 到得了的節點 `slow` 一定到得了。至於為什麼要多判一個 `fast->next`，是因為下一行要解兩層 `fast->next->next`，少判就會在偶數長度的無環串列上解到空指標。

### 方法二：hash set 記錄走過的節點 — O(n)／O(n)

把每個造訪過的節點指標丟進 `unordered_set`，一旦插不進去（代表來過）就是有環。思路最直白，代價是 O(n) 額外空間。

```cpp
// Time: O(n)   每個節點最多造訪一次，雜湊操作均攤 O(1)
// Space: O(n)  最壞情況存下所有節點指標
class Solution {
 public:
  bool hasCycle(ListNode *head) {
    unordered_set<ListNode *> visited;
    while (head) {
      if (!visited.insert(head).second) {
        return true;
      }
      head = head->next;
    }
    return false;
  }
};
```

> [!tip] `insert().second` 一次查找就問完「在不在」與「放進去」
> 寫成 `if (visited.contains(head)) return true; visited.insert(head);` 一樣對，但會算兩次雜湊、走兩次桶。`insert` 回傳 `pair<iterator, bool>`，`.second` 為 `false` 就代表元素本來就在裡面，等於免費拿到查詢結果。（`contains` 是 C++20 才有的，C++17 以前用 `count(head)` 或 `find(head) != visited.end()`。）

> [!note] set 裡放的是指標，不是值
> 判環問的是「有沒有回到同一個節點」，不是「有沒有出現同樣的值」。題目允許節點值重複（例如 `7 → 7 → 7 → 7` 無環），存 `int` 會全部誤判成有環，必須存 `ListNode *`。

## Related Problems

- [[0142-Linked-List-Cycle-II]] — 同一套 Floyd 的加強版，相遇後再從 head 與相遇點同速前進即可找到入環節點
- [[0287-Find-the-Duplicate-Number]] — 把陣列看成 `i → nums[i]` 的隱式鏈結串列，重複值就是環的入口，Floyd 原封不動搬過去
- [[0202-Happy-Number]] — 數字迭代形成的隱式串列，判斷會不會繞回舊值，快慢指針寫法一模一樣
- [[0876-Middle-of-the-Linked-List]] — 同款快慢指針，用在無環串列上就是找中點
- [[0143-Reorder-List]] — 快慢指針找中點的實戰，可以對照兩種停位寫法的差別
