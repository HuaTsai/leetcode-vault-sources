---
leetcode-id: 2
difficulty: medium
tags:
  - linked-list
  - math
  - grind-169
  - neetcode-150
memo: 迴圈條件要寫成 l1 或 l2 或 carry 三者取或，把進位一併當終止條件，否則 5 加 5 這種答案比兩串都長的情形會漏掉尾端的 1；取位數用 divmod 而非 if 判斷再減 10，除了換進位制能直接沿用，那個分支的成立機率接近五成、幾乎必然誤預測，實測反而比常數取模還慢
dg-publish: true
---

## Problem Description

You are given two non-empty linked lists representing two non-negative integers. The digits are stored in reverse order, and each of their nodes contains a single digit. Add the two numbers and return the sum as a linked list.

You may assume the two numbers do not contain any leading zero, except the number 0 itself.

Constraints:

- The number of nodes in each linked list is in the range `[1, 100]`.
- `0 <= Node.val <= 9`
- It is guaranteed that the list represents a number that does not have leading zeros.

## Solution

核心觀念：這題就是**小學直式加法**，唯一要想清楚的是「反序儲存」到底是麻煩還是禮物。答案是禮物——直式加法本來就從個位算起，而反序儲存剛好把個位放在 head，**沿著 `next` 走的方向正好就是進位傳播的方向**，一趟走完即可，完全不必反轉。

真正的細節只有兩個：進位要跨迴圈帶著走，以及**答案可能比兩串都長**（`[5] + [5] = [0,1]`），所以「還有沒有事情要做」不能只看兩串走完沒，還要看 `carry` 是不是清空了。

```txt
342 + 465 = 807

  l1:  2 → 4 → 3        (低位在前，讀作 342)
  l2:  5 → 6 → 4        (讀作 465)

  個位   2 + 5 + 0 =  7   → 寫 7，carry 0
  十位   4 + 6 + 0 = 10   → 寫 0，carry 1
  百位   3 + 4 + 1 =  8   → 寫 8，carry 0
  收尾   兩串皆空且 carry 0 → 結束

  ans: 7 → 0 → 8        (807)


為什麼 carry 必須進迴圈條件：

  l1:  5        l2:  5
  個位   5 + 5 + 0 = 10   → 寫 0，carry 1
  收尾   兩串已空，但 carry 還是 1
         → 必須再跑一輪，寫下最高位的 1

  ans: 0 → 1            (10)   ← 答案比兩串都長
```

### 方法一：Dummy Node + divmod — O(max(n,m))／O(1) 額外空間（推薦）

用 dummy node 消掉「第一個節點是誰」的特判，一趟走完。`while (l1 || l2 || carry)` 這個條件同時處理了三件事：兩串不等長、其中一串提早耗盡、以及最後可能多出來的進位。

```cpp
// Time: O(max(n, m))   每個節點各被讀一次，最多再多跑一輪處理尾端進位
// Space: O(1)          除了輸出的 max(n,m)+1 個節點之外沒有額外配置
class Solution {
 public:
  ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
    ListNode dummy;
    ListNode* cur = &dummy;
    int carry = 0;
    while (l1 || l2 || carry) {
      int sum = carry;
      if (l1) {
        sum += l1->val;
        l1 = l1->next;
      }
      if (l2) {
        sum += l2->val;
        l2 = l2->next;
      }
      carry = sum / 10;
      cur->next = new ListNode(sum % 10);
      cur = cur->next;
    }
    return dummy.next;
  }
};
```

> [!important] `carry` 必須是迴圈條件的一部分
> 寫成 `while (l1 || l2)` 然後在迴圈外補 `if (carry) cur->next = new ListNode(carry);` 一樣會過，但那是把同一段邏輯寫了兩次。把 `carry` 直接放進條件，「還有進位待處理」就與「還有節點待處理」享有完全相同的地位，收尾自然發生。
> 漏掉它的症狀很好認：`[5] + [5]` 得到 `[0]` 而不是 `[0,1]`，`[9,9] + [1]` 得到 `[0,0]` 而不是 `[0,0,1]`——**只有在最高位發生進位時才錯**，小測資很容易矇混過去。

> [!tip] 用 `sum / 10` 與 `sum % 10`，不要用 `if (sum >= 10) sum - 10`
> 寫成下面這樣是對的，因為 `sum ≤ 9 + 9 + 1 = 19`，進位永遠只有 0 或 1：
>
> ```cpp
> if (sum >= 10) { cur->next = new ListNode(sum - 10); carry = 1; }
> else           { cur->next = new ListNode(sum);      carry = 0; }
> ```
>
> 但它的正確性**依賴一個隱性的上界推理**。一旦節點裝的不是單一位數（大數表示法常讓每個節點存 4 位或 9 位），或換成別的進位制，`- 10` 立刻失效；`/ base`、`% base` 則原封不動照用——[[0067-Add-Binary]] 就只是把 10 換成 2。
>
> 反直覺的是**效能也不站在減法這邊**。`% 10` 的除數是編譯期常數，編譯器根本不發除法指令，而是用 magic number 換成一個乘法加兩個位移，全程無分支；反倒是 `if (sum >= 10)` 編出了真正的條件跳躍，而「兩個 0–9 隨機位數相加會進位」的機率是 45/100，正好落在分支預測器的最壞情況附近。實測純算術迴圈（i5-12500H、g++ 14.3 `-O2`、20000 組 × 100 位）：分支減法 9.77 ms、divmod 4.25 ms、無分支減法 3.03 ms——**分支版慢了一倍以上**。
>
> 若真的偏好減法，就寫成無分支形式 `carry = sum >= 10; ... sum - 10 * carry;`，它會編成 `setg` + `cmovle`。完整的機制與實測見 [[Micro-Optimization-Myths]]。

> [!note] `dummy` 開在 stack 上，不要 `new`
> 寫 `ListNode dummy;` 就好——`new ListNode()` 要記得 `delete`，忘了就洩漏。回傳 `dummy.next`（真正的第一個節點）；`&dummy` 在函式結束後就失效，**絕不能回傳**。同樣的套路見 [[0021-Merge-Two-Sorted-Lists]]。

### 方法二：就地重用輸入節點 — O(max(n,m))／O(1)

方法一的 O(1) 指的是**額外**空間，輸出本身仍要配置 `max(n,m)+1` 個節點。如果允許改寫輸入，可以把結果直接寫回 `l1`／`l2` 的既有節點，連 `new` 都幾乎省掉——只有「最高位進位」那一個節點是真的無中生有。

實務意義不在省記憶體，而在**省掉 allocator**：這題的成本大頭是 `new` 與指標追逐造成的 cache miss，算術只佔約三成。

```cpp
// Time: O(max(n, m))   同方法一
// Space: O(1)          最多只在最高位進位時配置 1 個節點，其餘重用輸入節點
class Solution {
 public:
  ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
    ListNode dummy;
    ListNode* cur = &dummy;
    int carry = 0;
    while (l1 || l2 || carry) {
      int sum = carry;
      if (l1) sum += l1->val;
      if (l2) sum += l2->val;
      carry = sum / 10;

      ListNode* node = l1 ? l1 : (l2 ? l2 : new ListNode());  // 先挑好要重用哪個
      if (l1) l1 = l1->next;                                  // 再各自前進
      if (l2) l2 = l2->next;

      node->val = sum % 10;
      cur->next = node;
      cur = node;
    }
    return dummy.next;
  }
};
```

> [!warning] 順序不能顛倒——先選節點，再前進指標
> `node = l1 ? l1 : ...` 必須寫在 `l1 = l1->next` 之前。一旦先前進，`l1` 已經指向下一個節點，重用的就是錯的那個，而且原本那個節點會從結果中消失。這也是為什麼這裡不能沿用方法一「邊加邊前進」的緊湊寫法，得把「累加」「選節點」「前進」拆成三步。

> [!note] 看起來該補一句 `cur->next = nullptr`，其實不必
> 重用的節點還帶著它在舊串列裡的 `next`，直覺上尾端會殘留一條指向舊資料的指標，所以很多寫法會在回傳前防禦性地補一句切斷。**但這裡可以證明它是多餘的**：迴圈只在 `l1`、`l2` 皆空且 `carry` 為 0 時結束，所以最後一輪取用的節點，必然是某一串走到底的那個尾節點（它的 `next` 本來就是 `nullptr`），或是為了最高位進位而新配置的節點（`next` 同樣是 `nullptr`）。中間各輪的舊 `next` 則都會被下一輪的 `cur->next = node` 覆寫掉。
> 我拿掉那行跑了 3009 筆測資（含 3000 筆長度各自隨機 1–100 的亂數）確認全過。留著無害，但知道為什麼安全，比多寫一行防禦更有價值。

## Related Problems

- [[0445-Add-Two-Numbers-II]] — 同一題但數字**正序**存放，低位在尾端、與進位方向相反，得先反轉或改用 stack 才能對齊
- [[0067-Add-Binary]] — 一模一樣的 carry 迴圈換成 base 2，divmod 版把 `/10`、`%10` 改成 `/2`、`%2` 就直接沿用，減法版則要重寫
- [[0066-Plus-One]] — 退化成只加 1 的進位傳播，唯一的長度變化發生在全 9 的情況
- [[0369-Plus-One-Linked-List]] — 0066 的鏈結串列版，正序存放所以同樣要處理方向問題
- [[0043-Multiply-Strings]] — 把進位思路推廣到乘法，carry 可以大到 8，`- 10` 那套徹底失效
- [[0021-Merge-Two-Sorted-Lists]] — dummy node 消掉頭節點特判的同一套路，同樣是「只重接指標」的一趟掃描
- [[Micro-Optimization-Myths]] — 本題 `sum % 10` 與 `if (sum >= 10)` 的取捨，背後是常數除法與分支誤預測的成本
