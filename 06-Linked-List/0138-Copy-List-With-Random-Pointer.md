---
leetcode-id: 138
difficulty: medium
tags:
  - hash
  - linked-list
  - neetcode-150
memo: 用舊節點到新節點的雜湊表跨過 random 的前向依賴；追問 O(1) 空間時把複製節點交錯插在原節點正後方，位置關係本身就是那張對照表
dg-publish: true
---

## Problem Description

A linked list of length `n` is given such that each node contains an additional random pointer, which could point to any node in the list, or `null`.

Construct a deep copy of the list. The deep copy should consist of exactly `n` brand new nodes, where each new node has its value set to the value of its corresponding original node. Both the `next` and `random` pointer of the new nodes should point to new nodes in the copied list such that the pointers in the original list and copied list represent the same list state. None of the pointers in the new list should point to nodes in the original list.

Return the head of the copied linked list.

## Solution

核心觀念：複製 `next` 毫無難度，一路往後建就好；真正卡住的是 `random` **會往後跳到還沒被 new 出來的節點**。這個前向依賴讓「邊走邊接」行不通，於是所有解法都在回答同一個問題——**怎麼從「舊節點」問出「它對應的新節點」**。

```txt
next   ：7 → 13 → 11 → 10 → 1
random ：7 ⇢ 10    13 ⇢ 7    11 ⇢ 1    10 ⇢ 11    1 ⇢ 7

一路往後複製，走到某個節點時：
  複製 7  → random 是 10，還沒建出來   ✗ 往後跳，接不了
  複製 13 → random 是 7，已經建好了    ✓ 往回跳，接得了

⇒ 單趟接不完 random。先把節點全部建出來，
  手上握著一張「舊節點 → 新節點」的對照表，第二趟再統一接線。
```

> [!important] 這題的資料結構就是一句話——舊節點到新節點的映射
> 難點不在鏈結串列操作，而在意識到「複製一個帶額外指標的結構」永遠需要 identity 的對照表。想清楚這件事，剩下的只是選什麼當 key／value。方法一直接用 `Node*` 對 `Node*`，方法三改用索引當中介，方法二則用「位置關係」取代整張表。

### 方法一：`unordered_map<Node*, Node*>` 兩趟 — O(n)／O(n)（推薦）

第一趟只 `new` 節點、不接任何指標，第二趟對照表已完整，`next` 和 `random` 一起接完。

```cpp
// Time: O(n)   兩趟各掃 n 個節點，雜湊查找平均 O(1)
// Space: O(n)  一張 n 筆的對照表
class Solution {
 public:
  Node* copyRandomList(Node* head) {
    unordered_map<Node*, Node*> mp;  // 舊節點 → 新節點

    // ① 只建節點，指標全部留空
    for (Node* p = head; p; p = p->next) {
      mp[p] = new Node(p->val);
    }

    // ② 對照表已完整，next 與 random 都能直接查出來
    for (Node* p = head; p; p = p->next) {
      mp[p]->next = p->next ? mp[p->next] : nullptr;
      mp[p]->random = p->random ? mp[p->random] : nullptr;
    }

    return head ? mp[head] : nullptr;
  }
};
```

> [!tip] 分兩趟是為了讓第二趟「查表必定命中」
> 只要接受「查不到就順手建一個」，就能壓成一趟。用一個 lambda 把「取得對應新節點，沒有就建」包起來，`mp[p]` 對不存在的 key 會 value-initialize 成 `nullptr`，正好當作「還沒建」的旗標：
>
> ```cpp
> auto get = [&](Node* p) -> Node* {
>   if (!p) return nullptr;
>   auto& q = mp[p];
>   if (!q) q = new Node(p->val);
>   return q;
> };
> for (Node* p = head; p; p = p->next) {
>   Node* c = get(p);
>   c->next = get(p->next);
>   c->random = get(p->random);
> }
> return get(head);
> ```
>
> 少掃一趟但多了「節點可能提前被建出來」的心智負擔，面試講兩趟版更好說明。

### 方法二：交錯編織，O(1) 額外空間 — O(n)／O(1)

不配置對照表，改把每個新節點**插在對應舊節點的正後方**。此時「舊節點的下一個就是它的複製」成為不變式，`p->random->next` 直接就是新節點該指的目標——位置關係取代了整張雜湊表。

```txt
原始:   A  →  B  →  C
編織:   A → A' → B → B' → C → C'
        A->random = C  ⇒  A'->random = A->random->next = C'
拆分:   A → B → C   與   A' → B' → C'
```

```cpp
// Time: O(n)   三趟線性掃描
// Space: O(1)  不計輸出，只借用原串列的 next 欄位
class Solution {
 public:
  Node* copyRandomList(Node* head) {
    if (!head) return nullptr;

    // ① 把複製節點編織進原串列
    for (Node* p = head; p; p = p->next->next) {
      Node* c = new Node(p->val);
      c->next = p->next;
      p->next = c;
    }

    // ② 靠「舊節點的下一個是它的複製」接 random
    for (Node* p = head; p; p = p->next->next) {
      p->next->random = p->random ? p->random->next : nullptr;
    }

    // ③ 拆成兩條，順手把原串列還原
    Node dummy(0);
    Node* t = &dummy;
    for (Node* p = head; p;) {
      Node* c = p->next;
      p->next = c->next;  // 還原原串列
      t = t->next = c;
      p = p->next;
    }

    return dummy.next;
  }
};
```

> [!warning] 三趟不能合併，尤其是 ① 和 ②
> 第 ② 步整段都依賴「**每一個**舊節點後面都已經跟著它的複製」。若在編織的同一趟就接 `random`，`p->random` 若指向尚未處理的後方節點，`p->random->next` 會拿到原串列的下一個舊節點，接出一堆指回原串列的錯誤指標——而題目明文禁止新串列指向舊節點。同理，③ 若和 ② 合併，拆到一半的串列會破壞第 ② 步還沒用到的位置關係。

> [!note] 拆分那步務必還原原串列
> `p->next = c->next;` 這句常被漏掉。LeetCode 判題端會檢查原串列有沒有被破壞，少了它原串列仍是編織後的狀態，會判錯。想確認自己的實作，寫測試時把原串列的 `(val, random 索引)` 快照下來，跑完再比一次。

> [!tip] 交錯編織這招在別題幾乎見不到，但它的「母題」到處都是
> 這個具體手法基本是 138 專屬——需要一個**線性且可暫時借用的指標欄位**，條件很苛。真正該記住的是背後兩條反覆出現的線索：
>
> 1. **用結構自身編碼額外資訊，換掉 O(n) 輔助表**——[[0289-Game-of-Life]] 用兩個 bit 在同一格存新舊兩代狀態、[[0448-Find-All-Numbers-Disappeared-in-an-Array]] 用正負號當「出現過」的標記、[[0287-Find-the-Duplicate-Number]] 用 Floyd 環偵測換掉 hash set。全是同一種交易：拿「暫時破壞再還原」換空間。
> 2. **串列的拆與合**——③ 那段「按位置把一條串列拆成兩條」跟 [[0328-Odd-Even-Linked-List]] 幾乎一模一樣；反過來的交錯合併則出現在 [[0143-Reorder-List]] 與 [[0021-Merge-Two-Sorted-Lists]]。這類指標搬移練熟了，① 和 ③ 寫起來就是肌肉記憶。

### 方法三：索引當中介 — O(n)／O(n)

把映射拆成兩層：`unordered_map<Node*, int>` 記錄舊節點的位序，`vector<Node*>` 用位序取新節點。本質與方法一等價，只是多繞一層索引。**沿著原串列遍歷**（而非遍歷 map）順序才確定，`next` 也就不必查表，直接取 `copies[i + 1]`。

```cpp
// Time: O(n)
// Space: O(n)  一張 map 加一個 vector，常數比方法一大
class Solution {
 public:
  Node* copyRandomList(Node* head) {
    unordered_map<Node*, int> idx;  // 舊節點 → 位序
    vector<Node*> copies;           // 位序 → 新節點

    for (Node* p = head; p; p = p->next) {
      idx[p] = copies.size();
      copies.push_back(new Node(p->val));
    }

    int i = 0;
    for (Node* p = head; p; p = p->next, ++i) {
      copies[i]->next = p->next ? copies[i + 1] : nullptr;
      copies[i]->random = p->random ? copies[idx.at(p->random)] : nullptr;
    }

    return head ? copies[0] : nullptr;
  }
};
```

> [!warning] 別一邊迭代 `unordered_map` 一邊用 `operator[]` 查它
> 若第二趟寫成 `for (auto& [node, i] : idx) { ... idx[node->random] ... }`，只要有一個 key 不存在，`operator[]` 就會**在迭代途中插入元素**，可能觸發 rehash 讓迭代器失效（UB）。本題所有 `random` 都指向串列內節點、key 必定存在，才僥倖沒事。真要在迭代中查同一個容器，用 `.at()`（查不到直接丟例外，而不是默默插入）；更好的做法是像上面一樣**改遍歷原串列**，順序確定又不碰 map 的結構。

> [!note] 為什麼實務上還是選方法一
> 索引版多了一次間接（`Node*` → `int` → `Node*`），程式碼更長、快取局部性更差，而且結構化綁定拿到的 `s` 就是位序，再寫一次 `idx[f]` 是白查一次雜湊。唯一真正需要索引的場合，是輸入本來就以 `[val, random_index]` 的陣列形式給你（LeetCode 內部正是這樣序列化這題）——那時 `vector` 版才是自然解。

## Related Problems

- [[0133-Clone-Graph]] — 同樣是「複製帶額外指標的結構」，一樣靠舊到新的對照表；但圖沒有線性順序，交錯編織這招用不上，正好凸顯方法一才是通解
- [[0328-Odd-Even-Linked-List]] — 按位置把一條串列拆成兩條，就是方法二第 ③ 步的獨立版
- [[0143-Reorder-List]] — 交錯合併兩條串列，是第 ③ 步的逆操作，指標搬移的手感相同
- [[0287-Find-the-Duplicate-Number]] — 同樣是「不用 O(n) 輔助結構」的追問，改用 Floyd 環偵測榨出 O(1) 空間
- [[0289-Game-of-Life]] — 另一種「原地把兩份資訊塞進同一個容器」，用 bit 編碼而非位置關係
- [[0146-LRU-Cache]] — 雜湊表與鏈結串列合作的另一經典，map 的 value 一樣是節點指標
