---
leetcode-id: 110
difficulty: easy
tags:
  - tree
  - depth-first-search
  - binary-tree
  - grind-169
  - neetcode-150
memo: 用 －1 當「已失衡」哨兵，把高度與答案塞進同一個 int 回傳值；剪枝必須寫在兩次 dfs 呼叫之間，寫成 l ＜ 0 ｜｜ r ＜ 0 時左右早就都算完了、一個節點都省不到，而拿到 l、r 後不先檢查負值就算 abs，－1 配 0 會得到 1 被當成平衡，四節點鏈就是反例
dg-publish: true
---

## Problem Description

Given a binary tree, determine if it is height-balanced.

A height-balanced binary tree is a binary tree in which the depth of the two subtrees of every node never differs by more than one.

## Solution

核心觀念：題目要的是「**每個**節點都滿足左右高度差 ≤ 1」，照字面做就是對每個節點各量一次左右子樹高度——但高度是 [[0104-Maximum-Depth-of-Binary-Tree]] 的後序回傳值，**子節點回報高度給我的那一刻，判斷失衡需要的兩個數字就已經在手上了**。所以一趟後序就夠。剩下的問題只是「怎麼把第二份資訊（失不失衡）帶上去」：合法高度恆 ≥ 0，於是 `-1` 這個值是空的，拿來當哨兵——**回傳非負代表高度，回傳 -1 代表這棵子樹底下已經失衡**。哨兵一旦出現就一路往上傳，中途的兄弟子樹連碰都不用碰。

```txt
        1              左子樹回報 -1 的那一刻，右邊整棵樹都不必走
       / \
      2   (一大棵樹)   dfs(5) = 1
     /                 dfs(4) = 2    |1 - 0| = 1 ok
    3                  dfs(3) = -1   |2 - 0| = 2 > 1  <- 在這裡失衡
   /                   dfs(2) = -1   左邊是 -1，直接往上傳
  4                    dfs(1) = -1   不再遞迴右子樹
 /
5
```

### 方法一：一趟後序，-1 當「已失衡」哨兵 — O(n)／O(h)（推薦）

```cpp
// Time: O(n)   每個節點至多造訪一次，提早失衡時更少
// Space: O(h)  遞迴堆疊深度＝樹高，平衡樹 O(log n)、退化成鏈 O(n)
class Solution {
 public:
  int dfs(TreeNode *root) {
    if (!root) {
      return 0;
    }
    int l = dfs(root->left);
    if (l < 0) {
      return -1;  // 左邊已失衡，右子樹整棵跳過
    }
    int r = dfs(root->right);
    if (r < 0) {
      return -1;
    }
    if (abs(l - r) > 1) {
      return -1;
    }
    return max(l, r) + 1;
  }

  bool isBalanced(TreeNode *root) {
    return dfs(root) >= 0;
  }
};
```

> [!important] `-1` 之所以能當哨兵，是因為合法高度用不到它
> 高度以節點數計，`dfs(nullptr) == 0` 是下界，任何真實高度都 ≥ 0——`-1` 不會和任何合法回傳值撞號，所以一個 `int` 就能同時載「高度」和「已失衡」兩種語意，不必回傳 `pair` 也不必開成員變數。這是樹遞迴很常見的一招，但**換個題目就未必安全**：[[0124-Binary-Tree-Maximum-Path-Sum]] 的路徑和可以是任何整數，就沒有這種「空出來的值」可以徵用。挑哨兵前先確認它落在合法值域之外，`INT_MIN` 更是不能亂用（`abs(INT_MIN)` 直接 UB）。

> [!warning] 拿到 `l`、`r` 後必須先檢查負值，直接算 `abs(l - r)` 會答錯
> 少了那兩行 `if (l < 0)`，`l = -1`（左邊已失衡）配上 `r = 0`（右邊是 null）會得到 `abs(-1 - 0) == 1`，不大於 1，於是回傳 `max(-1, 0) + 1 == 1`——**失衡訊號被吞掉，父節點還以為這是一棵高度 1 的正常子樹**。窮舉所有樹形實測：4 個節點的 14 種形狀裡就有 **8 種答錯**，最小反例是四節點鏈 `[1,null,1,null,1,null,1]`；20000 棵隨機樹裡錯 **35%**。哨兵的代價就是「每個讀到它的地方都要先認得它」，不能只在產生的地方寫對。

> [!warning] `if (l < 0 || r < 0 || abs(l - r) > 1)` 答案對，但一個節點都沒省
> 這是最容易誤以為有剪枝的寫法：`||` 確實會短路，但短路發生在 `l` 和 `r` **都已經遞迴算完之後**，右子樹早就走過了。實測隨機樹 `n = 5000`，這種寫法平均造訪 **10001 次**，而把 `return -1` 插在兩次 `dfs` 呼叫**之間**的版本只要 **23 次**。剪枝的位置是在兩次遞迴呼叫中間，不是在判斷式裡。

> [!tip] 剪枝省的是「發現失衡之後」，省不掉「發現之前」的下探
> 實測（`-O2`，visits ＝ `dfs` 進入次數）：
>
> | 測資 | 無剪枝 | 有剪枝 |
> |---|---|---|
> | 完美二元樹 h=20（平衡） | 2.79 ms／2097151 | 2.82 ms／2097151 |
> | 左邊四節點鏈失衡＋右邊 1M 節點 | 2.87 ms／2097159 | **0.0001 ms／8** |
> | 隨機樹 n=5000 ×200 棵平均 | 0.063 ms／10001 | **0.0001 ms／23** |
> | 斜鏈 n=5000 | 0.021 ms／10001 | 0.013 ms／5004 |
>
> 三件事：樹**真的平衡**時剪枝毫無收益（也毫無成本，2.79 vs 2.82 在誤差內），因為必須看完每個節點才能確定——**這題不可能比 O(n) 更好**。隨機樹幾乎一失衡就被逮到，快上三個數量級。而斜鏈那列說明剪枝的極限：失衡是**回程**才知道的，下探到底的 5000 層省不掉，只有回程路上的兄弟子樹省得掉。

### 方法二：自頂向下，對每個節點各量一次高度 — O(n·h)／O(h)

照題目定義硬做：檢查 root 平衡、再遞迴檢查兩棵子樹。

```cpp
// Time: O(n·h)  最壞 O(n²)（退化成鏈），平衡樹 O(n log n)
// Space: O(h)   遞迴堆疊深度＝樹高
class Solution {
 public:
  int depth(TreeNode *root) {
    return root ? max(depth(root->left), depth(root->right)) + 1 : 0;
  }

  bool isBalanced(TreeNode *root) {
    if (!root) {
      return true;
    }
    return abs(depth(root->left) - depth(root->right)) <= 1 &&
           isBalanced(root->left) && isBalanced(root->right);
  }
};
```

> [!warning] 慢在同一份高度被重算，而且它最慢的場合正好是方法一最快的場合
> 實測完美二元樹 h=20（`n ≈ 10⁶`）：`depth` 被呼叫 **3984 萬次、51.6 ms**，方法一是 **2.8 ms**，差約 18 倍——每個節點的高度被它的所有祖先各重算一遍，多出來的就是那個 `h`。值得注意的是這兩種寫法的難易場合**恰好相反**：平衡樹讓方法一無從剪枝、卻是方法二重算最兇的形狀；一失衡就被逮到的樹讓方法一秒殺、方法二反而因為 `&&` 短路也不算太慢（同一棵 1M 節點的失衡樹只要 6.0 ms）。和 [[0543-Diameter-of-Binary-Tree]] 的方法二是同一個教訓：**見到「每個節點都要問子樹一個量」，先想能不能塞進同一趟後序。**

## Related Problems

- [[0104-Maximum-Depth-of-Binary-Tree]] — 本題的 `dfs` 就是它，差別只在多徵用一個 `-1` 當失衡訊號
- [[0543-Diameter-of-Binary-Tree]] — 一樣後序回報高度順便算別的，但它把答案收在全域變數；本題示範另一種收集方式：塞進回傳值
- [[0124-Binary-Tree-Maximum-Path-Sum]] — 同一族後序骨架，但值域涵蓋所有整數，沒有空著的值可以拿來當哨兵
- [[0098-Validate-Binary-Search-Tree]] — 同樣是「每個節點都要成立」的性質，也能一發現違反就短路往上傳
- [[0108-Convert-Sorted-Array-to-Binary-Search-Tree]] — 反過來造一棵平衡樹，正好驗證這裡的平衡定義
