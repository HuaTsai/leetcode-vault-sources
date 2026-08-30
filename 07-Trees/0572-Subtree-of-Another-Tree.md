---
leetcode-id: 572
difficulty: easy
tags:
  - tree
  - depth-first-search
  - string
  - binary-tree
  - hash
  - grind-169
  - neetcode-150
memo: 走訪大樹每個節點各當一次起點呼叫 isSameTree，null 守衛 return p ＝＝ q 比的是指標不是值；先量出 subRoot 的高度當剪枝條件，只有高度恰好吻合的節點才值得比對，最壞測資實測從 150 萬次節點造訪掉到 1000 次；要拚 O（n＋m）就序列化後跑 KMP，但 null 一定要序列化成標記、分隔符一定要放在值的前面，否則 2 會匹配進 12 裡面
dg-publish: true
---

## Problem Description

Given the roots of two binary trees `root` and `subRoot`, return `true` if there is a subtree of `root` with the same structure and node values of `subRoot` and `false` otherwise.

A subtree of a binary tree `tree` is a tree that consists of a node in `tree` and all of this node's descendants. The tree `tree` could also be considered as a subtree of itself.

## Solution

核心觀念：`subRoot` 要成為子樹，必須從 `root` 的**某個節點開始，連同它底下所有後代**都一模一樣——不能只對上半截，也不能中途喊停。定義裡「and all of this node's descendants」就是在講這件事。於是最直接的解法是把 [[0100-Same-Tree]] 當成子程序：走訪 `root` 的每個節點，各問一次「以你為根，跟 `subRoot` 相同嗎」。**外層在選起點，內層在驗證**，兩件事分開想就不會亂。

```txt
root:      3            subRoot:   4
         /   \                    / \
        4     5                  1   2
       / \
      1   2

  以 3 為起點比 → 值就不同，false
  以 4 為起點比 → 4／1／2 全中，true ← 找到了

反例（LC example 2）：root 的 4 底下多掛了一個 0
        4                subRoot:   4
       / \                         / \
      1   2                       1   2
         /
        0
  上半截完全對得起來，但 subRoot 的 2 是葉子、root 的 2 還有孩子
  → 子樹必須連後代一起相同，false
```

這題掛 easy 是因為 `n, m ≤ 2000`，O(n × m) 的暴力就能過；但題目底下那句 follow-up「Could you solve it in O(n + m) time?」是不折不扣的 hard，要靠方法三的序列化 + 字串匹配才做得到。

### 方法一：每個節點都當一次起點 — O(n × m)／O(h)（推薦）

```cpp
// Time:  O(n × m)  n 個節點各當一次起點，單次 isSameTree 最多走 O(m)
// Space: O(h)      遞迴堆疊，斜樹退化成 O(n)
class Solution {
 public:
  bool isSameTree(TreeNode *p, TreeNode *q) {
    if (!p || !q) {
      return p == q;
    }
    return p->val == q->val && isSameTree(p->left, q->left) &&
           isSameTree(p->right, q->right);
  }

  bool isSubtree(TreeNode *root, TreeNode *subRoot) {
    if (!root) {
      return false;
    }
    return isSameTree(root, subRoot) || isSubtree(root->left, subRoot) ||
           isSubtree(root->right, subRoot);
  }
};
```

> [!important] 兩層遞迴在做完全不同的事
> 外層 `isSubtree` 負責**挑起點**，內層 `isSameTree` 負責**驗證**。看清楚這件事就不會寫出「比到一半發現不合，想從當前位置繼續往下比」的錯誤結構——比對一旦失敗，就是換一個起點從頭再驗一次，中途的部分匹配一點都不能留。`||` 的短路求值順便幫你做到「找到就停」。

> [!tip] `return p == q` 比的是指標，不是值
> 能走到那行代表 `p`、`q` 至少一個是 `nullptr`，於是「兩個指標相等」唯一的可能就是兩個都是 `nullptr`，一行同時講完 true（都是 null）和 false（只有一個是 null）。這個守衛的完整推導在 [[0100-Same-Tree]]，重點是它**依賴上一行的 `if (!p || !q)`**，搬到函式最前面就變成在問「是不是同一顆記憶體」，整題就錯了。

> [!warning] 外層的 `!root` 要回 false，不要跟著寫成 true
> 內層 `isSameTree` 的 null 是「兩邊同時走完，算相符」；外層 `isSubtree` 的 null 是「大樹這條路走到底都沒找到」，語意相反。本題約束保證 `subRoot` 至少有一個節點，所以不必處理空樣式；若題目改成允許 `subRoot` 為空，空樹是任何樹的子樹，得在最前面先 `if (!subRoot) return true;`。

用 BFS 逐節點掃描配同一個 `isSameTree` 也完全等價，只是這題不需要任何層的資訊，寫成遞迴會短很多。

### 方法二：先量高度再比對 — 最壞 O(n × m)／O(h)，實務接近 O(n)

方法一的浪費在於：`root` 裡有一大票節點的**高度根本對不上** `subRoot`，卻還是各付了一次 `isSameTree` 的錢。既然子樹要求「連後代一起完全相同」，那**高度必須相等**是個免費的必要條件。用一趟後序把每個節點的高度算出來，只在高度吻合時才真的去比。

```cpp
// Time:  最壞仍 O(n × m)，但只有高度恰好等於 subRoot 的節點會進 isSameTree
// Space: O(h)  一趟後序的遞迴堆疊
class Solution {
 public:
  bool isSubtree(TreeNode *root, TreeNode *subRoot) {
    bool found = false;
    probe(root, subRoot, height(subRoot), found);
    return found;
  }

 private:
  bool isSameTree(TreeNode *p, TreeNode *q) {
    if (!p || !q) {
      return p == q;
    }
    return p->val == q->val && isSameTree(p->left, q->left) &&
           isSameTree(p->right, q->right);
  }

  int height(TreeNode *node) {
    return node ? 1 + max(height(node->left), height(node->right)) : 0;
  }

  // 後序回傳自己的高度，順手在高度吻合時才付一次 isSameTree 的錢
  int probe(TreeNode *node, TreeNode *subRoot, int target, bool &found) {
    if (!node) {
      return 0;
    }
    int h = 1 + max(probe(node->left, subRoot, target, found),
                    probe(node->right, subRoot, target, found));
    if (!found && h == target && isSameTree(node, subRoot)) {
      found = true;
    }
    return h;
  }
};
```

> [!important] 高度一定要用「一趟後序回傳」，不能在每個節點上現算
> 若寫成走訪時對每個節點呼叫一次 `height(node)`，光是算高度就變成 O(n × h)，斜樹直接 O(n²)，剪枝省下來的全部賠回去。這裡的 `probe` 是「後序回傳一個數值、順路在回傳途中更新答案」的骨架，跟 [[0543-Diameter-of-Binary-Tree]]、[[0124-Binary-Tree-Maximum-Path-Sum]] 是同一個模子。

> [!tip] 剪枝有多有效——實測
> 最壞測資：`root` 是長度 2000、值全為 1 的左偏鏈，`subRoot` 是長度 1000 的同款鏈但**最底端的值不同**（所以答案必為 false，每次比對都得走到底才發現不對）。統計進入 `isSameTree` 的節點造訪次數：
>
> | 作法 | isSameTree 節點造訪 | 時間 |
> | --- | --- | --- |
> | 方法一（不剪枝） | 1,501,499 | 14.14 ms |
> | 方法二（高度剪枝） | 1,000 | 0.03 ms |
>
> 理論最壞複雜度仍是 O(n × m)（高度相同的候選節點不見得只有一個），但同高度的節點彼此不可能是祖孫、子樹互不重疊，候選數被 n／(h+1) 壓著，實務上幾乎就是一趟走完。

### 方法三：序列化 + KMP — O(n + m)／O(n + m)

follow-up 要的線性解。把兩棵樹各自序列化成一個字串，「`subRoot` 是不是 `root` 的子樹」就退化成「B 是不是 A 的子字串」，用 KMP 一次掃完。

```cpp
// Time:  O(n + m)  兩趟序列化 + 一次 KMP
// Space: O(n + m)  兩個序列化字串與前綴函數
class Solution {
 public:
  bool isSubtree(TreeNode *root, TreeNode *subRoot) {
    string text, pat;
    serialize(root, text);
    serialize(subRoot, pat);
    string s = pat + '\x01' + text;  // 分隔符必須是兩邊都不會出現的字元
    vector<int> pi = prefixFunction(s);
    int m = pat.size();
    for (int i = m + 1; i < (int)s.size(); ++i) {
      if (pi[i] == m) {
        return true;
      }
    }
    return false;
  }

 private:
  // 前序走訪，null 也要留下標記，且逗號放在值的「前面」
  void serialize(TreeNode *node, string &s) {
    if (!node) {
      s += ",#";
      return;
    }
    s += ',';
    s += to_string(node->val);
    serialize(node->left, s);
    serialize(node->right, s);
  }

  vector<int> prefixFunction(const string &s) {
    vector<int> pi(s.size(), 0);
    for (int i = 1; i < (int)s.size(); ++i) {
      int j = pi[i - 1];
      while (j > 0 && s[i] != s[j]) {
        j = pi[j - 1];
      }
      if (s[i] == s[j]) {
        ++j;
      }
      pi[i] = j;
    }
    return pi;
  }
};
```

> [!warning] 序列化有兩個各自能單獨害死你的坑
> **坑一：null 沒有序列化成標記。** 跳過 null 的前序序列不足以決定樹形：
>
> ```txt
> root:  1        subRoot:  2
>       /                  /
>      2                  3
>       \
>        3
> ```
>
> 不記 null 的前序是 `1,2,3` 與 `2,3`，子字串成立 → 誤判 true，但 `subRoot` 顯然不是子樹（root 的 2 是靠**右孩子**接到 3 的）。記了 null 之後變成 `,1,2,#,3,#,#,#` 與 `,2,3,#,#,#`，正確回 false。
>
> **坑二：分隔符放在值的後面。** 就算 null 有標記，寫成 `s += to_string(val) + ','` 一樣會爆：`root = [12]`、`subRoot = [2]` 的序列是 `12,#,#,` 與 `2,#,#,`，後者是前者的子字串 → 誤判 true。**逗號要放在值的前面**（或值的兩側都補邊界），讓每個 token 都有左界，多位數才不會被咬掉一截。這兩個錯誤版本我都實測過，各自在對應的測資上翻車。

> [!note] 也可以改用樹雜湊
> 把每棵子樹遞迴算成一個雜湊值（例如 `hash(node) = f(val, hash(left), hash(right))`），再問 `subRoot` 的雜湊有沒有出現在 `root` 的雜湊集合裡，一樣是 O(n + m)。代價是要處理碰撞——面試講得出這條路就夠了，真要寫還是 KMP 穩，因為它是確定性的。

## Related Problems

- [[0100-Same-Tree]] — 本題直接把它當子程序呼叫；`return p == q` 那個指標守衛的完整推導、以及「前序序列要含 null 標記」的陷阱都在那篇
- [[0104-Maximum-Depth-of-Binary-Tree]] — 方法二用的 `height` 就是那題的答案原封不動搬過來當剪枝條件
- [[0543-Diameter-of-Binary-Tree]] — 同一個「後序回傳一個數值、順路在回傳途中更新答案」的骨架，對照方法二的 `probe`
- [[0101-Symmetric-Tree]] — 雙指標同步遞迴的另一個變形，下降時左右交叉配對
- [[0028-Find-the-Index-of-the-First-Occurrence-in-a-String]] — 方法三序列化之後剩下的那個純字串問題，KMP 的正主
- [[0652-Find-Duplicate-Subtrees]] — 「把子樹壓成字串／雜湊再丟進 hash map」的正統應用，序列化那套在那題是主線而非替代解

[[String-Matching]] — KMP 前綴函數的完整推導、拼接分隔符為什麼不能省，以及字串雜湊的 anti-hash 風險
