---
leetcode-id: 199
difficulty: medium
tags:
  - tree
  - depth-first-search
  - breadth-first-search
  - binary-tree
  - grind-169
  - neetcode-150
memo: 先右後左推入佇列，q.front() 就是該層最右，等於把層內「我是不是最後一個」的判斷編碼進推入順序；陷阱是每層最右不等於從 root 一路往右，右子缺位時要由左子樹的分支補位
dg-publish: true
---

## Problem Description

Given the `root` of a binary tree, imagine yourself standing on the right side of it, return the values of the nodes you can see ordered from top to bottom.

## Solution

核心觀念：右視圖看到的是**每一層最右邊的節點**，而不是「從 root 一路往右走」。把題目讀成「按層分組，每組取最後一個」之後，它就是 [[0102-Binary-Tree-Level-Order-Traversal]] 改一行的直接應用；剩下的全部功夫都花在「怎麼挑出層內最後一個」這件小事上。

```txt
        1              第 0 層 → 1
      /   \
     2     3           第 1 層 → 3
      \      \
       5      4        第 2 層 → 4
      /
     6                 第 3 層 → 6   ← 3 和 4 都沒有小孩，這層由左邊的分支補位

答案 = [1,3,4,6]
```

> [!important] 「每層最右」不等於「一路往右」
> 上圖第 3 層的代表是 6，它掛在 2 的右子 5 底下——右側那條鏈在 4 就斷了，視線於是穿過去看到更左邊的分支。隨機測資實測：5000 棵隨機樹裡有 4598 棵會被「從 root 一路往右」解錯。這條錯誤思路在完全二元樹上會矇對，所以特別難靠自己的測資發現。

### 方法一：BFS 剝層，剝之前取 `q.back()` — O(n)／O(n)（推薦）

沿用層序走訪的骨架：`int n = q.size()` 凍結當層長度、內圈把整層剝乾淨。差別只在不必收集整層的值，進入迴圈時直接問「這層最右邊是誰」。

```cpp
// Time: O(n)   每個節點進出佇列各一次
// Space: O(n)  佇列最多裝下一整層，滿二元樹最後一層約 n/2 個節點
class Solution {
 public:
  vector<int> rightSideView(TreeNode* root) {
    if (!root) {
      return {};
    }
    vector<int> ans;
    queue<TreeNode *> q;
    q.push(root);
    while (!q.empty()) {
      ans.push_back(q.back()->val);
      int n = q.size();
      while (n--) {
        auto fn = q.front();
        q.pop();
        if (fn->left) {
          q.push(fn->left);
        }
        if (fn->right) {
          q.push(fn->right);
        }
      }
    }
    return ans;
  }
};
```

> [!tip] `q.back()` 必須在剝層**之前**取
> 進入 `while` 的那一刻，佇列裡恰好是完整的一層：`front()` 是最左、`back()` 是最右。一旦開始 pop / push，`back()` 就變成下一層正在長出來的節點了。這跟 `int n = q.size()` 得先凍結是同一個道理——**所有對「這一層」的提問都必須在剝之前問完**，這是 [[0102-Binary-Tree-Level-Order-Traversal]] 那顆釘子的另一種用法。

### 方法二：反轉推入順序，取 `q.front()` — O(n)／O(n)

既然「最右」不好取，就讓它變成「最左」：先 push `right` 再 push `left`。

```cpp
// Time: O(n)   每個節點進出佇列各一次
// Space: O(n)  同方法一，佇列最多裝下一整層
class Solution {
 public:
  vector<int> rightSideView(TreeNode* root) {
    if (!root) {
      return {};
    }
    queue<TreeNode *> q;
    q.push(root);
    vector<int> ans;
    while (!q.empty()) {
      ans.push_back(q.front()->val);
      int n = q.size();
      while (n--) {
        auto fn = q.front();
        q.pop();
        if (fn->right) {
          q.push(fn->right);
        }
        if (fn->left) {
          q.push(fn->left);
        }
      }
    }
    return ans;
  }
};
```

```txt
先右後左推入 → 佇列裡每一層都是由右到左

q = [1]      front=1 ✓
q = [3,2]    front=3 ✓      ← 3 是右子，先進來
q = [4,5]    front=4 ✓
q = [6]      front=6 ✓
```

> [!important] 推入順序反轉之後，「最後一個」就變成「第一個」
> 這不是省幾個字的小聰明：它把「挑出層內某個位置的元素」這個需求，**編碼進佇列的順序本身**，內圈於是完全不必知道自己是第幾個。代價是佇列裡的層變成由右到左，之後若要改成「每層平均」「之字形」這類需要層內原始順序的題目，得記得順序已經被翻過。想保住與 0102 共用的直覺就用方法一，想讓內圈更乾淨就用這版，兩者成本相同（見下方實測）。

### 方法三：DFS 先右後左 ＋ `depth == ans.size()` — O(n)／O(h)

不用佇列也行：前序走訪改成「根 → 右 → 左」，那麼每一層**第一個被踏到**的節點就是該層最右邊的節點，而「第一次踏進第 depth 層」的判定條件正好是 `depth == ans.size()`。

```cpp
// Time: O(n)   每個節點恰好造訪一次
// Space: O(h)  遞迴堆疊深度＝樹高（不計答案本身）
class Solution {
 public:
  vector<int> rightSideView(TreeNode* root) {
    dfs(root, 0);
    return ans;
  }

 private:
  vector<int> ans;

  void dfs(TreeNode* node, int depth) {
    if (!node) {
      return;
    }
    if (depth == (int)ans.size()) {
      ans.push_back(node->val);
    }
    dfs(node->right, depth + 1);
    dfs(node->left, depth + 1);
  }
};
```

> [!warning] 遞迴順序與判定條件兩處都容易寫反
>
> - **必須先遞迴 `right` 再 `left`**。寫反就成了左視圖，而且在右鏈完整的樹上照樣輸出看起來合理的答案，不見得會被小測資抓到。
> - **判定要用 `==`，不是 `>=`**。前序保證深度一格一格往下長，`ans.size()` 永遠是「已經記錄過的層數」，所以「深度等於現有層數」就是「第一次踏進這層」，不需要另外維護最大深度或事後回填。`ans.size()` 是 `size_t`，跟 `int depth` 直接比會觸發 sign-compare 警告，記得轉型。

> [!note] 實測：三種寫法的差別在常數，而且方向取決於樹形
> 2000 節點、`g++ -O2`、cachegrind 數指令數（重複 2000 次）：
>
> | 寫法                          | I refs（隨機樹） | I refs（2000 深左斜樹） |
> | ----------------------------- | ---------------- | ----------------------- |
> | 方法二（先右後左 ＋ `front`） | 97.0M            | 224.6M                  |
> | 方法一（`q.back()`）          | 97.2M            | 237.0M                  |
> | 內圈判斷 `i == n - 1`         | 109.7M           | 245.6M                  |
> | 方法三（DFS）                 | 115.8M           | 118.8M                  |
>
> 三件事：方法一與方法二差 0.15%，**等價，選哪個純看可讀性**（D1 misses 換 4 個 seed 量互有輸贏，是樹形抽樣雜訊）；把取值改寫成內圈的 `if (i == n - 1)` 要多付 13% 指令，因為那個比較每個節點都得做一次，而前兩種寫法每層只問一次；DFS 在隨機樹上多 19%（葉節點那約 n/2 次踩空呼叫照樣建棧幀），但在鏈狀樹上只有 BFS 的一半——BFS 每層都要付一次外圈成本，2000 層就付 2000 次。
>
> 本題 n ≤ 100，這些常數全無實務意義；列出來是為了確認「反轉推入順序」沒有偷偷變貴，以及 DFS 真正的賣點是 O(h) 空間而非速度。

## Related Problems

- [[0102-Binary-Tree-Level-Order-Traversal]] — 同一副剝層骨架，本題等於它「每層只留最後一個」的特例；`int n = q.size()` 的陷阱兩題共用
- [[0513-Find-Bottom-Left-Tree-Value]] — 方法二的鏡像：改成先左後右推入，最後一層的 `q.front()` 就是答案，連分層都可以省掉
- [[0104-Maximum-Depth-of-Binary-Tree]] — 方法三的 `depth` 參數在那題就是答案本身，是「深度當狀態往下傳」的最小練習
- [[1448-Count-Good-Nodes-In-Binary-Tree]] — 同樣是前序 DFS 帶狀態往下傳，差別在那題傳的是路徑最大值、這題傳的是深度
- [[0103-Binary-Tree-Zigzag-Level-Order-Traversal]] — 對照組：那題需要層內的原始順序，方法二那種「把順序翻掉」的技巧就不能用
