---
leetcode-id: 1448
difficulty: medium
tags:
  - tree
  - depth-first-search
  - breadth-first-search
  - binary-tree
  - neetcode-150
memo: DFS 沿路徑把目前最大值往下傳，等值也算 good 要用大於等於；初值直接給 root 自己的值，根節點就不必特例或哨兵
dg-publish: true
---

## Problem Description

Given a binary tree `root`, a node X in the tree is named good if in the path from root to X there are no nodes with a value greater than X.

Return the number of good nodes in the binary tree.

## Solution

一個節點是不是 good，**只跟 root 到它這條路徑上的最大值有關**，跟兄弟節點、跟自己的子樹都無關。所以不必對每個節點回頭往上掃一遍祖先（那是 O(n·h)），只要在前序 DFS 往下走的時候把「路徑最大值」當參數一路帶下去，每個節點就能 O(1) 判斷。

```txt
        3            pathMax=3   3>=3 good，往下傳 3
       / \
      1   4          1<3  bad，傳 3     4>=3 good，傳 4
     /   / \
    3   1   5        3>=3 good    1<4 bad    5>=4 good

答案 = 4
```

> [!important]
> 這是「資訊由上往下傳」的典型題——狀態隨遞迴往下累積，答案在走訪過程中順手數完。對照 [[0124-Binary-Tree-Maximum-Path-Sum]] 是反方向的「由下往上回傳」。判斷資訊該往哪個方向流，樹的題目就解掉一半。

> [!warning]
> 判斷要用 `>=` 而不是 `>`。題目說的是路徑上沒有 **greater than** X 的節點，所以祖先跟自己等值時仍然算 good。`[2,2,2]` 的答案是 3 不是 1，這是本題最常見的 off-by-one。

### 方法一 — 前序 DFS 帶著路徑最大值往下傳 — O(n)／O(h)

```cpp
// Time: O(n)，每個節點恰好走訪一次
// Space: O(h)，遞迴堆疊；平衡樹 O(log n)，斜樹退化成 O(n)
class Solution {
 public:
  int goodNodes(TreeNode *root) {
    ans = 0;
    dfs(root, root->val);
    return ans;
  }

 private:
  void dfs(TreeNode *node, int pathMax) {
    if (node->val >= pathMax) {
      ++ans;
      pathMax = node->val;  // by value，只影響這層以下的遞迴
    }
    if (node->left) dfs(node->left, pathMax);
    if (node->right) dfs(node->right, pathMax);
  }

  int ans;
};
```

`pathMax` 是 by value 傳入的，所以在函式裡直接改它不會污染呼叫端，右子樹拿到的仍是進入這層時的值，不需要額外的暫存變數。

> [!tip]
> **「root 一定是 good」不必用哨兵表達。** root 沒有祖先所以必然是 good，這個特例有三種寫法：
>
> - `dfs(root, nullptr)`——用「不存在」當哨兵，語意誠實，但 `pathMax` 因此得是 `TreeNode *`，同時承載存在性與值兩種資訊（見方法三）。
> - `dfs(root, INT_MIN)`——`INT_MIN` 不是隨手塞的 magic number，它是 `max` 的單位元（`max(∅) = -∞`），不靠題目值域 `[-10^4, 10^4]` 僥倖成立。但讀者仍要多驗一步「這值真的在值域外嗎」。
> - `dfs(root, root->val)`——root 跟自己比，`>=` 恆真。特例不是被處理掉，而是被編碼進一般情況裡，遞迴內零分支、零哨兵，連 `<climits>` 都不用。
>
> 三者都對，第三種概念負擔最小。

### 方法二 — 顯式 stack 迭代 — O(n)／O(h)

把 `(節點, 進入該節點時的路徑最大值)` 成對壓進 stack，就不需要遞迴。斜樹時 h = n，遞迴版有爆堆疊的風險，這版沒有。

```cpp
// Time: O(n)
// Space: O(h)，stack 最多同時存一條路徑上的分支
class Solution {
 public:
  int goodNodes(TreeNode *root) {
    int ans = 0;
    stack<pair<TreeNode *, int>> st;
    st.emplace(root, root->val);
    while (!st.empty()) {
      auto [node, pathMax] = st.top();
      st.pop();
      if (node->val >= pathMax) {
        ++ans;
        pathMax = node->val;
      }
      if (node->left) st.emplace(node->left, pathMax);
      if (node->right) st.emplace(node->right, pathMax);
    }
    return ans;
  }
};
```

把 `stack` 換成 `queue` 就變成 BFS，答案一樣——因為每個節點的判斷只依賴自己那條路徑，走訪順序完全不影響結果。

### 方法三 — 傳祖先節點指標的變體 — O(n)／O(h)

與方法一同一個演算法，只是用 `nullptr` 當「路徑上還沒有節點」的哨兵，`pathMax` 因而是指標：

```cpp
// Time: O(n)
// Space: O(h)
class Solution {
 public:
  int goodNodes(TreeNode *root) {
    ans = 0;
    dfs(root, nullptr);
    return ans;
  }

 private:
  void dfs(TreeNode *node, TreeNode *pathMax) {
    TreeNode *next = nullptr;
    if (!pathMax || node->val >= pathMax->val) {
      ++ans;
      next = node;
    } else {
      next = pathMax;
    }
    if (node->left) dfs(node->left, next);
    if (node->right) dfs(node->right, next);
  }

  int ans;
};
```

> [!note]
> **實測：這個改寫不是效能問題，是表達力問題。** 用 10 萬節點隨機樹重複走訪 100 次（cachegrind + 計時）：
>
> | 寫法 | I refs | D1 misses | wall time |
> | --- | --- | --- | --- |
> | 方法三（指標） | 211.8M | 12.86M | 241／236 ms |
> | 方法一（`int`） | 158.9M | 12.84M | 238／235 ms |
>
> 指標版每個節點多一次 `!pathMax` 判斷與一次 `pathMax->val` 解參考，指令數多了 25%——但**時間沒有差別**。兩個原因：那個祖先就在遞迴路徑上、剛被走訪過，一直待在 L1，所以 cache miss 幾乎一樣；而這題的瓶頸本來就是隨機樹的 pointer chasing（1280 萬次 D1 miss），CPU 等記憶體的空檔足以吞掉多出來的指令。
>
> 所以選方法一的理由是「`pathMax` 只用得到 `->val`，用 `TreeNode *` 等於拿指標表達 `optional<int>`，讀者要多想一層」，不是「比較快」。

## Related Problems

- [[0098-Validate-Binary-Search-Tree]] — 同樣把約束往下傳，只是傳的是上下界區間而非單一最大值
- [[0104-Maximum-Depth-of-Binary-Tree]] — 最小的「狀態往下傳」練習，傳的是深度
- [[0124-Binary-Tree-Maximum-Path-Sum]] — 對照組，資訊由下往上回傳而非往下傳
- [[0113-Path-Sum-II]] — 一樣是前序累積路徑狀態，但要保留整條路徑而不只是聚合值
