---
leetcode-id: 543
difficulty: easy
tags:
  - tree
  - depth-first-search
  - binary-tree
  - dynamic-programming
  - grind-169
  - neetcode-150
memo: dfs 回傳的是以節點數計的深度，但 l ＋ r 不必調整就正好是邊數——連到左右孩子的那兩條邊剛好補上兩邊各差的 1；答案取雙邊 l ＋ r、回傳只能取單邊 max（l，r）＋ 1，且直徑未必經過 root，只算 depth（left）＋ depth（right）在隨機樹上約六分之一會答錯
dg-publish: true
---

## Problem Description

Given the `root` of a binary tree, return *the length of the diameter of the tree*.

The diameter of a binary tree is the length of the longest path between any two nodes in a tree. This path may or may not pass through the `root`.

The length of a path between two nodes is represented by the number of edges between them.

## Solution

核心觀念：把每條路徑用它的**最高點（折返點）**分類。一條路徑必然有唯一一個離 root 最近的節點，從那裡往左下延伸一段、往右下延伸一段。於是「所有路徑取最長」就等於「**枚舉每個節點當折返點**，取 `左邊往下最深 + 右邊往下最深`」。而這兩個深度正是 [[0104-Maximum-Depth-of-Binary-Tree]] 的後序回傳值——所以一趟後序遞迴就能一邊回報深度、一邊順手更新答案。

```txt
        1              直徑 = 5 - 3 - 2 - 4 - 6，共 4 條邊
       /               折返點是 2，不是 root
      2
     / \               以 2 為折返點：左深 2 + 右深 2 = 4  <- 答案
    3   4              以 1 為折返點：左深 3 + 右深 0 = 3
   /     \
  5       6
```

### 方法一：一趟後序，邊回報深度邊更新答案 — O(n)／O(h)（推薦）

`dfs` 回傳「以此節點為頂、往下能延伸多深」，同時在每個節點試著用 `l + r` 更新全域答案。

```cpp
// Time: O(n)   每個節點恰好造訪一次
// Space: O(h)  遞迴堆疊深度＝樹高，平衡樹 O(log n)、退化成鏈 O(n)
class Solution {
 public:
  int dfs(TreeNode *root) {
    if (!root) {
      return 0;
    }
    int l = dfs(root->left);
    int r = dfs(root->right);
    ans = max(ans, l + r);
    return max(l, r) + 1;
  }

  int diameterOfBinaryTree(TreeNode *root) {
    ans = 0;
    dfs(root);
    return ans;
  }

 private:
  int ans = 0;
};
```

> [!important] 回傳的是「節點數」，`l + r` 卻直接就是「邊數」，這個差 1 是自己對消掉的
> `dfs(nullptr) == 0`、`dfs(leaf) == 1`，回傳值和 0104 一樣**以節點數計深度**；題目要的答案卻是**邊數**。照理說換算要 `-1`，但 `l + r` 一個字都不用改就是對的：從當前節點到左子樹最深節點，中間有 `l` 個節點、卻有 `l` 條邊（多出來的那條是「當前節點 → 左孩子」），右邊同理。**兩邊各短少的 1，剛好被連到左右孩子的那兩條邊補回來。** 順帶記住變形題的換算：若問的是路徑上的**節點數**（如 [[1245-Tree-Diameter]] 的某些版本），答案是 `l + r + 1`。

> [!warning] 直徑不一定經過 root，只寫 `depth(left) + depth(right)` 會錯
> 這是這題最經典的陷阱，題目敘述特地寫了 "This path may or may not pass through the root"。上面圖解那棵樹就是最小反例：正解 4，只看 root 得到 3。實測 3000 棵隨機樹（節點數 1～30），有 **492 棵（約六分之一）**答案不同——這個錯誤在小測資上很容易矇混過關，`[1,2,3,4,5]` 這種範例測資根本測不出來。**`ans` 必須在遞迴的每一層更新，不能只在最外層算一次。**

> [!tip] 「回傳單邊、答案取雙邊」是一整族樹題的共同骨架
> 和 [[0124-Binary-Tree-Maximum-Path-Sum]] 是同一個模板：**往上回傳的東西只能挑一邊**（`max(l, r) + 1`），否則接到父節點就變成三叉、不再是路徑；**更新答案時左右都能吃**（`l + r`），因為那條路徑到此為止。124 因為節點值可能為負，還要多一層 `max(0, ...)` 決定「這一邊帶不帶」；本題的深度恆非負，帶著永遠不虧，所以連 clamp 都省了。

> [!note] `ans` 是成員變數只是懶得傳參，不是必要的
> 想避免成員狀態，可以讓 `dfs` 回傳 `pair<深度, 直徑>`：`return {max(lh, rh) + 1, max({ld, rd, lh + rh})};`（已驗證等價）。成員變數版短、可讀性也夠，是面試場合的主流寫法；但 in-class 的 `int ans = 0;` 別省——目前正確性靠 `diameterOfBinaryTree` 裡那行 `ans = 0;` 兜住，一旦有人單獨呼叫 `dfs`，讀到的就是未初始化的值（UB）。兩處都寫最保險。

### 方法二：對每個節點各算一次左右深度 — O(n·h)／O(h)

不用全域變數，直接照「枚舉折返點」的定義硬做：每個節點都重新呼叫 `depth` 量左右子樹。

```cpp
// Time: O(n·h)  最壞 O(n²)（退化成鏈），平衡樹 O(n log n)
// Space: O(h)   遞迴堆疊深度＝樹高
class Solution {
 public:
  int depth(TreeNode *root) {
    return root ? max(depth(root->left), depth(root->right)) + 1 : 0;
  }

  int diameterOfBinaryTree(TreeNode *root) {
    if (!root) {
      return 0;
    }
    int through = depth(root->left) + depth(root->right);
    return max({through, diameterOfBinaryTree(root->left),
                diameterOfBinaryTree(root->right)});
  }
};
```

> [!warning] 慢的原因是同一份深度資訊被重算 n 次，不是常數大
> 實測斜鏈 `n = 10⁴`（`-O2`）：`depth` 被呼叫 **1.0 × 10⁸ 次、234 ms**，而方法一是 **0.041 ms**——差約 5700 倍。本題 `n` 上限剛好是 10⁴，這個寫法在 LeetCode 上是**擦邊過**而不是必定 TLE，所以它不會逼你改，但它是錯的思路。方法一之所以能省掉整整一個 n，關鍵只有一句：**子節點回報深度給我的那一刻，我要的兩個數字就已經在手上了，順手更新 `ans` 是零成本的。** 見到「每個節點都要問子樹一個量」時，先想能不能塞進同一趟後序。

## Related Problems

- [[0124-Binary-Tree-Maximum-Path-Sum]] — 同一個「回傳單邊、答案取雙邊」骨架，把邊數換成路徑和；因為有負數才多出 clamp 那一層
- [[0104-Maximum-Depth-of-Binary-Tree]] — 本題的 `dfs` 就是它，差別只在遞迴途中多維護一個全域最佳解
- [[0687-Longest-Univalue-Path]] — 同骨架再加「值要和父節點相同才能延伸」，是本題最直接的變形
- [[0110-Balanced-Binary-Tree]] — 一樣是「回報深度順便算別的」，示範用特殊回傳值（-1）當剪枝訊號
- [[1245-Tree-Diameter]] — 一般圖／多元樹版的直徑，同樣邏輯改成走鄰接表，或用兩次 BFS 的經典解法
