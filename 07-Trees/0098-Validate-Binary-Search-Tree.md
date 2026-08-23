---
leetcode-id: 98
difficulty: medium
tags:
  - tree
  - depth-first-search
  - binary-search-tree
  - grind-169
  - neetcode-150
memo: BST 的條件是對整條祖先鏈成立、不是對父節點成立，所以要一路把上下界收窄；界線用 nullptr 代表無界，拿 INT_MIN／INT_MAX 當哨兵會在節點值剛好是極值時誤判，中序版的 prev 同理要用指標而不是 INT_MIN
dg-publish: true
---

## Problem Description

Given the `root` of a binary tree, determine if it is avalid binary search tree (BST).

A ***valid BST*** is defined as follows:

- The left subtree of a node contains only nodes with keys strictly less than the node's key.
- The right subtree of a node contains only nodes with keys strictly greater than the node's key.
- Both the left and right subtrees must also be binary search trees.

## Solution

核心觀念：BST 的限制是**祖先鏈上的所有節點一起施加**的，不是只跟父節點比。往左走時上界收成父節點的值、往右走時下界收成父節點的值，每個節點都被夾在一個開區間裡——這是**方法一**。另一個等價說法是 **BST 的中序走訪必定嚴格遞增**，於是驗證退化成「檢查中序序列有沒有逆序」——這是**方法二**。

```txt
        5                每一對父子單獨看都合法（3 < 6、7 > 6）
      /   \              但 3 位在 5 的右子樹裡，必須 > 5
     4     6             → 不是 BST
          / \
         3   7           上下界視角：走到 6 時區間是 (5, ∞)
                                    走到 3 時區間是 (5, 6)，3 不在裡面

        中序視角：4 5 3 6 7 —— 5 之後掉回 3，出現逆序
```

### 方法一：上下界遞迴 — O(n)／O(h)（推薦）

把「還沒有界線」表達成 `nullptr`，而不是找一個數值來當哨兵。

```cpp
// Time: O(n)   每個節點恰好造訪一次
// Space: O(h)  遞迴堆疊深度＝樹高，平衡樹 O(log n)、退化成鏈 O(n)
class Solution {
  bool dfs(TreeNode* node, TreeNode* low, TreeNode* high) {
    if (!node) {
      return true;
    }
    if (low && node->val <= low->val) {
      return false;
    }
    if (high && node->val >= high->val) {
      return false;
    }
    return dfs(node->left, low, node) && dfs(node->right, node, high);
  }

 public:
  bool isValidBST(TreeNode* root) { return dfs(root, nullptr, nullptr); }
};
```

> [!important] 條件要對「整條祖先鏈」成立，不是對父節點成立
> 只檢查 `root->left->val < root->val && root->right->val > root->val` 再往下遞迴，是這題最經典的錯法：`[5,4,6,null,null,3,7]` 的每一對父子都合法，它會回傳 `true`，但 3 隔代違反了 5。**遞迴要往下傳的不是「父節點」而是「累積下來的區間」**——往左走換上界、往右走換下界，兩邊各只動一個。

> [!warning] 別拿 `INT_MIN`／`INT_MAX` 當「無界」的哨兵
> 題目的值域正是 `[-2^31, 2^31-1]`，哨兵和合法節點值撞在一起。寫成 `dfs(root, INT_MIN, INT_MAX)` 配 `val <= low || val >= high`，在 `[2147483647]`、`[-2147483648]` 這種單一節點樹上就會回傳 `false`。**只要「不存在」需要一個表示法，就別去值域裡挑，用 `nullptr`（或 `optional<int>`）給它一個獨立的表示。** 改用 `long long` 邊界確實能過，但那是靠「節點值只到 int」這個題目保證把哨兵推到值域外，換一題值域是 64-bit 就再度失效；順帶一提要寫 `long long` 不是 `long`——LeetCode 是 LP64 才碰巧沒事，Windows 的 `long` 仍是 32-bit。

> [!note] `optional<int>` 也可以，只是比較胖
> `dfs(node, optional<int> low, optional<int> high)` 同樣安全，語意上更貼近「界線是個值」。差別是每層多兩個 8-byte 參數在堆疊上搬，指標版直接借用既有節點、不多花空間。

### 方法二：中序遍歷嚴格遞增 — O(n)／O(h)

只需要記住**前一個**中序節點，發現逆序就能立刻收工。骨架本身（`cur` 一路往左壓進 stack、彈出時才處理）是通用的，整理在 [[Tree-Traversal-Iterative]]。

```cpp
// Time: O(n)   每個節點進出堆疊各一次，逆序時提早 return
// Space: O(h)  顯式堆疊最多裝下一條根到葉的路徑
class Solution {
 public:
  bool isValidBST(TreeNode* root) {
    stack<TreeNode*> st;
    TreeNode *cur = root, *prev = nullptr;
    while (cur || !st.empty()) {
      while (cur) {
        st.push(cur);
        cur = cur->left;
      }
      cur = st.top();
      st.pop();
      if (prev && prev->val >= cur->val) {
        return false;
      }
      prev = cur;
      cur = cur->right;
    }
    return true;
  }
};
```

> [!tip] 「BST ⇔ 中序嚴格遞增」是一整組題目的共用工具
> 驗證只是它的一種用法：[[0230-Kth-Smallest-Element-in-a-BST]] 是數到第 k 個就 return、[[0783-Minimum-Distance-Between-BST-Nodes]] 是沿路取相鄰差的最小值——骨架都是這個迴圈，變的只有 `prev` 和 `cur` 之間要做什麼。注意是**嚴格**遞增，所以判斷式是 `>=` 而不是 `>`，重複值不合法。

> [!warning] `prev` 用指標而不是 `int prev = INT_MIN`
> 和方法一同一個坑：節點值剛好是 `INT_MIN` 時第一次比較就誤判。用 `TreeNode* prev = nullptr` 讓「還沒有前驅」有自己的表示。另外若寫成遞迴版、把 `prev` 放成 member 變數，記得每次進入 `isValidBST` 要重設——LeetCode 每筆測資會新建一個 `Solution` 所以線上不會炸，但一旦宣告成 `static` 就會被上一筆測資的殘值毒到。

> [!note] 要 O(1) 空間就是 Morris 中序
> 用左子樹最右節點的空 `right` 指標暫時接回目前節點來取代堆疊，走過後再拆掉。省下的 O(h) 空間換來的是「遍歷途中樹是被改動過的」，多執行緒或不允許修改輸入時不能用，面試也很少要求到這一步。

## Related Problems

- [[0230-Kth-Smallest-Element-in-a-BST]] — 同一個中序骨架，把「檢查有沒有逆序」換成「數到第 k 個就 return」
- [[0094-Binary-Tree-Inorder-Traversal]] — 方法二那段迭代中序的原型題，先把模板寫熟這題就是加一行比較
- [[Tree-Traversal-Iterative]] — 前中後序三種迭代骨架的整理，方法二用的是其中的中序模板
- [[0235-Lowest-Common-Ancestor-of-a-Binary-Search-Tree]] — 一樣利用「往下走時區間持續收窄」，差別在拿它導航而不是驗證
- [[0700-Search-in-a-Binary-Search-Tree]] — 最單純的 BST 性質應用，每層只往一邊走所以是 O(h)
