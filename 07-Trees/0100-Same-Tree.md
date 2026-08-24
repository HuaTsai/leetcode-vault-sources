---
leetcode-id: 100
difficulty: easy
tags:
  - tree
  - depth-first-search
  - breadth-first-search
  - binary-tree
  - grind-169
  - neetcode-150
memo: 兩棵樹同步下降，null 守衛寫成「至少一個是 null 就回傳 p ＝＝ q」即可一次涵蓋兩者皆 null 與恰好一個 null，因為那是指標比較；複雜度是 O（min（n，m））而非 O（n），＆＆ 短路一撞到差異就砍掉整條右子樹，另外別拿不含 null 標記的前序序列判等，只有左孩子與只有右孩子的兩棵樹序列同為 1、2 卻不是同一棵樹
dg-publish: true
---

## Problem Description

Given the roots of two binary trees `p` and `q`, write a function to check if they are the same or not.

Two binary trees are considered the same if they are structurally identical, and the nodes have the same value.

## Solution

核心觀念：前面幾題的遞迴都只帶一個指標往下走，這題帶**兩個**。走法完全對稱——`p` 往左、`q` 就往左，兩邊永遠停在「同一個位置」上，於是「兩棵樹相同」被拆成每一組同位置節點都要滿足的三件事：**要嘛同時是 null，要嘛同時非 null 且值相同、且左右子樹遞迴也相同**。難的不是遞迴本身，是把 null 的組合處理乾淨。

```txt
p:    1          q:    1              p:    1          q:    1
    /   \            /   \                /                  \
   2     3          2     3              2                    2

   同位置節點兩兩相符 → true        值的集合一樣（1、2）但結構不同 → false

同步下降：(p, q) → (p->left, q->left) 與 (p->right, q->right) 成對往下
```

### 方法一：雙指標同步遞迴 — O(min(n, m))／O(min(h1, h2))（推薦）

```cpp
// Time:  O(min(n, m))  只要一邊先撞到 null 或值不同就短路，走訪量由較小的那棵樹封頂
// Space: O(min(h1, h2)) 遞迴堆疊深度，斜樹退化成 O(min(n, m))
class Solution {
 public:
  bool isSameTree(TreeNode* p, TreeNode* q) {
    if (!p || !q) {
      return p == q;
    }
    return p->val == q->val && isSameTree(p->left, q->left) &&
           isSameTree(p->right, q->right);
  }
};
```

> [!important] `return p == q` 比的是**指標**，不是值
> 這行看起來像在比兩個節點相不相等，其實不是——能走到這裡代表 `p`、`q` 至少一個是 `nullptr`，所以「兩個指標相等」唯一可能的情況就是**兩個都是 `nullptr`**，剛好把 true（同時為 null）和 false（恰好一個為 null）兩種結果一次講完。它成立的前提是上一行的守衛，**換到函式最前面就會變成在問「這兩個是不是同一顆記憶體」，整題就錯了。**

> [!tip] 複雜度是 O(min(n, m))，不是 O(n)
> `&&` 會短路：只要 `p->val != q->val`，右邊兩個遞迴呼叫**完全不會執行**，整條子樹被砍掉。而且只要走到一邊 null、另一邊非 null 就立刻回 false，所以能走訪的節點數被較小的那棵樹封頂。兩棵樹真的相同時才會退化成走完全部——那也是唯一必須看完每個節點的情況。

> [!warning] 別想用「比較兩棵樹的前序序列」偷懶
> 直覺上「把兩棵樹各印一次前序、字串一樣就是同一棵」很誘人，但跳過 null 的遍歷序列**不足以決定樹形**：`[1,2]`（只有左孩子）和 `[1,null,2]`（只有右孩子）的前序序列都是 `1、2`，卻不是同一棵樹。序列法要成立，**null 也必須被序列化成一個標記**（例如印成 `#`）；這正是 [[0572-Subtree-of-Another-Tree]] 用字串比對解時最容易踩的坑。

### 方法二：把 null 守衛分段寫開 — O(min(n, m))／O(min(h1, h2))

同一套遞迴，只是不用指標比較的技巧，把判斷一段一段講明白。初學時這樣寫更容易確認自己沒漏 case：

```cpp
// Time:  O(min(n, m))
// Space: O(min(h1, h2))
class Solution {
 public:
  bool isSameTree(TreeNode* p, TreeNode* q) {
    if (!p && !q) {
      return true;
    }
    if (!p || !q) {
      return false;
    }
    return p->val == q->val && isSameTree(p->left, q->left) &&
           isSameTree(p->right, q->right);
  }
};
```

> [!note] 第二個守衛不需要寫成 `(!p && q) || (p && !q)`
> 直覺上會想把「恰好一個是 null」完整拼出來，但**上一行已經把「兩個都是 null」排除掉了**：能執行到第二行的狀態只剩「兩個都非 null」「只有 p 是 null」「只有 q 是 null」三種，在這個位置「至少一個是 null」就**等價於**「恰好一個是 null」。多寫的那兩個 `&&` 是在重新檢查一件上一行已經保證的事。這是讀 null 守衛時值得養成的習慣：**先問「走到這一行時，還可能是什麼狀態」，再決定條件要寫多細。**

### 方法三：迭代版，用堆疊存「待比較的配對」 — O(min(n, m))／O(min(h1, h2))

遞迴堆疊裡真正在傳遞的就是一組 `(a, b)` 配對，把它顯式搬到自己的 `stack` 上就得到迭代版，好處是極斜的樹上不吃系統遞迴堆疊。

```cpp
// Time:  O(min(n, m))
// Space: O(min(h1, h2)) 堆疊只留下沿途未處理的兄弟，每層最多一組，和遞迴版同級
class Solution {
 public:
  bool isSameTree(TreeNode* p, TreeNode* q) {
    stack<pair<TreeNode *, TreeNode *>> st;
    st.emplace(p, q);
    while (!st.empty()) {
      auto [a, b] = st.top();
      st.pop();
      if (!a || !b) {
        if (a != b) {
          return false;
        }
        continue;
      }
      if (a->val != b->val) {
        return false;
      }
      st.emplace(a->left, b->left);
      st.emplace(a->right, b->right);
    }
    return true;
  }
};
```

> [!tip] 配對一定要**成對**入堆疊，不能各推各的
> 常見的錯誤寫法是開兩個 stack、`p` 一個 `q` 一個，然後各自 push 非 null 的孩子。那會壞掉：跳過 null 之後兩邊的推入順序就不再對齊，`[1,2]` 和 `[1,null,2]` 會被判成相同。**`pair` 的意義就是把「這兩個節點處於同一個位置」這件事釘死**，null 也照樣入堆疊，由取出時的守衛負責判。

> [!important] 為什麼是 `stack` 而不是 `queue`？答案是空間
> 把 `stack` 換成 `queue`（`top` 改 `front`）**答案完全一樣**——這題不需要層的概念、也不依賴走訪順序，只是要窮舉所有同位置配對，先深先廣都對。差別在容器的峰值大小，而且差得很誇張：
>
> ```txt
> 樹形                    n      h    stack 峰值    queue 峰值
> 完美二元樹          65535     16           17         65536
> 左斜鏈             100000 100000            2             3
> ```
>
> `stack` 是深度優先，鑽到底就回頭，任何時刻只留下**沿途每層未處理的那個兄弟**，所以峰值 = O(h)；`queue` 是廣度優先，會把**整整一層**同時攤在容器裡，完美二元樹最後一層就佔了約 n/2 個節點、連同 null 配對達到 n+1。**同一份程式碼只換一個容器，空間就從 O(h) 變成 O(n)。**

> [!tip] 這題和 [[0104-Maximum-Depth-of-Binary-Tree]] 的取捨方向剛好相反
> 那題的答案**本身就是層數**，`queue` 的 O(n) 空間是換取「一次剝一層」這個結構、非付不可；這題根本不在乎層，於是沒有任何理由去付那筆錢。**選容器不是看習慣，是看題目要不要「層」這個資訊**——不要的話，`stack` 白拿 O(h)。

> [!note] 迭代版換掉的是系統堆疊，不是空間複雜度
> `stack` 版和遞迴版同為 O(h)，改寫**不會**變省。理由是遞迴用的系統堆疊只有幾 MB 且爆掉直接 crash，而 `stack` 配在堆積上、容量大得多——極斜的樹（h = n = 10 萬）正是遞迴會爆而它不會的場合。

## Related Problems

- [[0101-Symmetric-Tree]] — 同一套雙指標同步遞迴，只差在下降方式改成交叉（左配右、右配左），等於「拿一棵樹和自己的鏡像做 isSameTree」
- [[0572-Subtree-of-Another-Tree]] — 直接把本題當子程序，在大樹每個節點上呼叫一次；也是「前序序列要含 null 標記」那個陷阱的正主
- [[0104-Maximum-Depth-of-Binary-Tree]] — 同一組 DFS 骨架，但那題的 null 要回傳一個值（0），守衛只能寫在進入點，對照本題的 null 只需回傳 bool
- [[0226-Invert-Binary-Tree]] — 也是兩兩對應地看左右子樹，但它是就地改寫而非比較
- [[0951-Flip-Equivalent-Binary-Trees]] — 本題的放寬版，允許任意節點交換左右子樹，遞迴時要多試一組交叉配對
