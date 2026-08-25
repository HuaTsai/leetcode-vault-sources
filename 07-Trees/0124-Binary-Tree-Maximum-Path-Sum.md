---
leetcode-id: 124
difficulty: hard
tags:
  - tree
  - dynamic-programming
  - depth-first-search
  - binary-tree
  - grind-169
  - neetcode-150
memo: 一個節點要回答兩個不同的問題，往上回傳的路徑只能挑單邊、更新答案時左右都能吃，所以 return 絕不能寫成 v4；另一個地雷是 dfs（left）與 dfs（right）各被呼叫兩次，複雜度會從 O（n）炸成 O（2ⁿ），務必先存成區域變數
dg-publish: true
---

## Problem Description

A path in a binary tree is a sequence of nodes where each pair of adjacent nodes in the sequence has an edge connecting them. A node can only appear in the sequence at most once. Note that the path does not need to pass through the root.

The path sum of a path is the sum of the node's values in the path.

Given the `root` of a binary tree, return the maximum path sum of any non-empty path.

## Solution

核心觀念：難點在於**一個節點要同時回答兩個不同的問題，而且兩個答案不一樣**。

1. **往上回傳**：以我為端點、往下延伸的最大路徑和。這條路徑之後還要接到父節點，所以**只能挑左右其中一邊**。
2. **更新全域答案**：以我為最高點（折返點）的最大路徑和。這條路徑到我為止就不往上了，所以**左右可以都吃**。

```txt
        -10
        /   \
       9     20
            /   \
          15     7

以 20 為折返點： 15 -> 20 -> 7      = 42   <- 這是答案，但它「不能往上接」
20 回傳給 -10 ： 20 + max(15, 7)   = 35   <- 接上去之後才還是一條路徑

若把 42 回傳給 -10，-10 底下就分成兩叉了，那不是路徑
```

### 方法一：四種形狀直接列舉 — O(n)／O(h)（推薦）

令 `L = dfs(root->left)`、`R = dfs(root->right)`（兩者都是「往下單邊延伸」的值）。以 `root` 為最高點的路徑只有四種形狀，全部列出來：

```txt
v1 = val              只有自己
v2 = val + L          自己 + 左邊那條
v3 = val + R          自己 + 右邊那條
v4 = val + L + R      自己 + 左右都吃（在 root 折返）
```

`ans` 從四個裡挑最大，回傳值只能從**前三個**裡挑。

```cpp
// Time: O(n)   每個節點恰好造訪一次，L 與 R 各只算一次
// Space: O(h)  遞迴堆疊深度＝樹高，平衡樹 O(log n)、退化成鏈 O(n)
class Solution {
 public:
  int dfs(TreeNode *root) {
    if (!root) {
      return 0;
    }
    int l = dfs(root->left);
    int r = dfs(root->right);
    int v1 = root->val;
    int v2 = root->val + l;
    int v3 = root->val + r;
    int v4 = root->val + l + r;
    ans = max(ans, max(v1, max(v2, max(v3, v4))));
    return max(v1, max(v2, v3));
  }

  int maxPathSum(TreeNode *root) {
    ans = root->val;
    dfs(root);
    return ans;
  }

 private:
  int ans;
};
```

> [!important] 回傳值和答案是兩個不同的東西，`return` 絕不能是 `v4`
> `v4` 是「在 root 折返」的路徑，它已經用掉 root 的左右兩個方向，再往上接就會在 root 變成三叉——那是子樹，不是路徑。所以 `v4` 只能貢獻給 `ans`，永遠不能被回傳。這個「**回傳單邊、答案取雙邊**」的骨架在 [[0543-Diameter-of-Binary-Tree]]、[[0687-Longest-Univalue-Path]] 完全一樣，只是把「和」換成「長度」。

> [!warning] `dfs(root->left)` 寫兩次會讓複雜度從 O(n) 炸成指數
> 若把 `v2`／`v4` 寫成 `root->val + dfs(root->left)` 這種形式，左右子樹各被算**兩次**，每個節點展開 4 條遞迴。平衡樹 `T(n) = 4T(n/2)` → **O(n²)**，退化成鏈 `T(n) = 2T(n-1)` → **O(2ⁿ)**。實測（`-O2`）：滿二元樹 n = 16383 要 357,913,941 次呼叫／402 ms，n = 262143 要 916 億次／102 秒；長度 30 的鏈就已經 42.9 億次／4 秒。本題 n 上限 3 × 10⁴，兩種樹形都必 TLE。**修法只是把兩次呼叫存成 `l`、`r` 兩個區域變數**，一個字都不用多改——遞迴裡重複呼叫是看不見的指數炸彈，養成「子樹的結果只算一次、先存起來」的習慣。

> [!warning] `ans` 不能初始化成 0
> 題目保證路徑**非空**，全負數的樹（例如 `[-3]`）答案是 -3 而不是 0。初值寫 `root->val`（題目保證 `n >= 1`）或 `INT_MIN` 都可以，寫 0 會錯。

### 方法二：把「要不要這個孩子」clamp 成 0 — O(n)／O(h)

題解最常見的寫法，短很多，代價是那個憑空冒出來的 `0` 需要解釋。

```cpp
// Time: O(n)   同樣每個節點一次
// Space: O(h)  遞迴堆疊深度＝樹高
class Solution {
 public:
  int dfs(TreeNode *root) {
    if (!root) {
      return 0;
    }
    int l = max(0, dfs(root->left));
    int r = max(0, dfs(root->right));
    ans = max(ans, root->val + l + r);
    return root->val + max(l, r);
  }

  int maxPathSum(TreeNode *root) {
    ans = root->val;
    dfs(root);
    return ans;
  }

 private:
  int ans;
};
```

**那個 `0` 不是新東西，它就是方法一的 `v1`。** `max(0, L)` 讀作「左邊那條路帶著划算就帶、不划算就當成 0（不帶）」，而「不帶」正是 `v1` 那條「只取自己」的候選。以下證明兩者嚴格等價。令 `L = dfs(left)`、`R = dfs(right)`，`l = max(0, L)`、`r = max(0, R)`。

**更新 `ans` 的部分**——先把 `val` 提出來：

```txt
方法一  max(v1, v2, v3, v4)
      = max(val, val + L, val + R, val + L + R)
      = val + max(0, L, R, L + R)

方法二  val + l + r
      = val + max(0, L) + max(0, R)
```

所以只要證 `max(0, L, R, L+R) = max(0, L) + max(0, R)`，依 L、R 的正負分四種情況窮舉：

```txt
     L          R      | max(0, L, R, L+R) | max(0,L) + max(0,R) |
  ---------------------+-------------------+---------------------+
   L >  0     R >  0   |  L + R            |  L + R              | 四者中 L+R 最大
   L >  0     R <= 0   |  L                |  L + 0  = L         | R <= 0 故 L >= L+R
   L <= 0     R >  0   |  R                |  0 + R  = R         | 與上一列對稱
   L <= 0     R <= 0   |  0                |  0 + 0  = 0         | 其餘三者皆 <= 0
```

四格全部相等，得證。

**回傳值的部分**——同樣提出 `val`，再用 `max` 的結合律：

```txt
方法一  max(v1, v2, v3)
      = max(val, val + L, val + R)
      = val + max(0, L, R)
      = val + max( max(0, L), max(0, R) )
      = val + max(l, r)

方法二  val + max(l, r)
```

> [!tip] 為什麼可以少列舉一半？因為兩個孩子的決定互不影響
> `max(0, L, R, L+R)` 表面上是在四種組合裡挑一個，但實際上「要不要帶左邊」和「要不要帶右邊」是**兩個獨立的決定**——帶左邊划不划算，跟右邊是什麼完全無關。獨立的決定可以各自最佳化再相加，於是四選一塌縮成 `max(0,L) + max(0,R)` 兩個各自二選一。方法一把四種形狀攤開來看得見在窮舉什麼，方法二把同一件事壓縮成「每個孩子各自 clamp」；**選看得懂的那個寫，別為了短而寫**。

> [!note] `if (!root) return 0;` 這裡回傳 0 剛好安全，別改成 `INT_MIN`
> 空子樹回傳 0 在方法一會讓 `v2` 退化成 `val`（等於 `v1`，重複但無害），在方法二則被 `max(0, ...)` 直接吸收掉。若為了「表示不存在」改成 `INT_MIN`，`root->val + INT_MIN` 立刻整數溢位——**這個 base case 回傳的不是「無限差」，而是「這一邊不貢獻」**。

## Related Problems

- [[0543-Diameter-of-Binary-Tree]] — 同一個「回傳單邊、答案取雙邊」骨架，把路徑和換成邊數；沒有負數所以連 clamp 都不用
- [[0687-Longest-Univalue-Path]] — 同骨架再加上「值要跟父節點相同才能延伸」的條件，是本題最直接的變形
- [[1372-Longest-ZigZag-Path-in-a-Binary-Tree]] — 回傳值得拆成左右兩個方向，把「一個節點該回報什麼」推到更難的一步
- [[0104-Maximum-Depth-of-Binary-Tree]] — 最單純的後序回報，本題等於在它上面多加一層「途中維護全域最佳解」
- [[0110-Balanced-Binary-Tree]] — 一樣邊回報邊更新全域狀態，並示範用特殊回傳值（-1）當剪枝訊號
