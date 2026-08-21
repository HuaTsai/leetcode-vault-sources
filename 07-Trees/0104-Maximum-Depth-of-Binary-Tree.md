---
leetcode-id: 104
difficulty: easy
tags:
  - tree
  - binary-tree
  - depth-first-search
  - breadth-first-search
  - grind-169
  - neetcode-150
memo: 深度以「節點數」計，null 回傳 0 讓 base case 只能寫在函式進入點，而 max（左，右）＋1 必須等子樹回報、是嚴格後序；BFS 版的關鍵是進內層迴圈前先把當層節點數凍結成 n，直接拿 q.size() 當條件會邊 pop 邊 push 而多算層數
dg-publish: true
---

## Problem Description

Given the `root` of a binary tree, return its maximum depth.

A binary tree's **maximum depth** is the number of nodes along the longest path from the root node down to the farthest leaf node.

## Solution

核心觀念：這題的「深度」以**節點數**計——空樹是 0、單一節點是 1。兩種寫法對應兩種看樹的角度：**遞迴**看的是「一個節點的深度 = 較深的那棵子樹 + 自己這一層」，**BFS** 看的是「一次剝掉一整層，剝了幾次就是幾層」。

```txt
        3                第 1 層：3
      /   \              第 2 層：9、20
     9     20            第 3 層：15、7
          /  \
        15    7          答案 = 3

遞迴：depth(3) = max(depth(9), depth(20)) + 1 = max(1, 2) + 1 = 3
BFS ：把整層一次清空，清了 3 次
```

### 方法一：遞迴 DFS — O(n)／O(h)（推薦）

```cpp
// Time: O(n)   每個節點恰好造訪一次
// Space: O(h)  遞迴堆疊深度＝樹高，平衡樹 O(log n)、退化成鏈 O(n)
class Solution {
 public:
  int maxDepth(TreeNode* root) {
    if (!root) {
      return 0;
    }
    return max(maxDepth(root->left), maxDepth(root->right)) + 1;
  }
};
```

> [!important] `if (!root) return 0;` 在這題是 base case，不是防呆
> 同樣是「進入點擋 null」，[[0226-Invert-Binary-Tree]] 那邊擋不擋、在哪擋都可以，因為 null 不必回傳任何東西；這題的 null 要回傳一個**值**（0），整條遞迴式全靠它收尾。若改成在呼叫端擋（`if (root->left) ...`），你得在每個呼叫點手動補上「沒有孩子時算 0」，程式碼會又長又容易漏。**null 需要回傳值時，守衛就只能寫在進入點。**

> [!tip] 這題是嚴格後序，順序不能像 226 那樣亂調
> `max(左, 右) + 1` 必須先拿到兩棵子樹的答案才能算自己——資訊是由下往上匯集的。226 的 `swap` 不看子樹狀態，所以前序後序皆可。這是樹遞迴的兩大類型：**「當下就能做完的事」順序自由，「要等子樹回報的事」只能後序。** 之後的 0110、0543 都屬於後者。

> [!note] 不必為了「優化」加葉節點特判
> `if (!root->left && !root->right) return 1;` 不會讓它變快，只是在每個節點多跑一個分支，然後把同樣的 0 從子層搬到本層算而已。

### 方法二：BFS 層序 — O(n)／O(n)

改成一層一層剝：進迴圈前先記下當層有幾個節點，把這幾個全部彈出並把孩子排進佇列，佇列被清空幾輪就是幾層。內層用 `while (n--)` 而不是 `for (int i = 0; i < n; ++i)`——索引從頭到尾沒被用到，取了名字反而誘導讀者去找它在哪裡被用。

```cpp
// Time: O(n)   每個節點進出佇列各一次
// Space: O(n)  佇列最多裝下一整層，滿二元樹最後一層約 n/2 個節點
class Solution {
 public:
  int maxDepth(TreeNode* root) {
    if (!root) {
      return 0;
    }
    queue<TreeNode *> q;
    q.push(root);
    int ans = 0;
    while (!q.empty()) {
      ++ans;
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

> [!warning] 層的大小一定要先存進 `n`，不能拿 `q.size()` 當迴圈條件
> 寫成 `for (size_t i = 0; i < q.size(); ++i)` 會壞掉：迴圈進行中一邊 pop 一邊 push，`q.size()` 是浮動的，內層會吃掉下一層的節點。陰險的是它在滿二元樹、單鏈這類規則樹形上**碰巧會給出正確答案**，隨機測資也要跑上千棵才踩得到；`[1,2,3,null,4,5,6,null,null,null,null,null,7]` 就是一個反例，正確答案 4 但它算出 5。`int n = q.size();` 這一行的意義就是**把「當層有幾個」凍結下來**，別省。

> [!tip] 這題的 BFS 沒有提早結束的機會，111 才有
> 求**最大**深度一定要走完每個節點，BFS 相對遞迴沒有任何時間優勢，唯一的價值是不吃遞迴堆疊（極斜的樹上遞迴 O(h) = O(n) 有爆堆疊風險，而 BFS 每層只有 1 個節點、實際只用 O(1)）。反過來在 [[0111-Minimum-Depth-of-Binary-Tree]]，BFS 碰到第一個葉節點就能直接 return，那才是它真正贏遞迴的場合。

> [!warning] 別把 `max` 換成 `min` 就當作 111 的解
> 樹是 `1 → 2`（只有左孩子）時，右邊 null 回傳 0，`min(1, 0) + 1 = 1`，但最短路徑必須走到**葉節點**，答案是 2。最小深度得特判「只有一邊有孩子」的情況，改一個字是不夠的。

## Related Problems

- [[0111-Minimum-Depth-of-Binary-Tree]] — 同一個模板但不能只把 max 換 min，單邊 null 會壓垮答案；也是 BFS 唯一能提早 return 的場合
- [[0226-Invert-Binary-Tree]] — 同一組 DFS／BFS 骨架，但它順序自由，正好是這題「嚴格後序」的對照組
- [[0110-Balanced-Binary-Tree]] — 直接拿本題當子程序，再加上「回傳 -1 表示已失衡」的剪枝，避免重複算深度
- [[0543-Diameter-of-Binary-Tree]] — 一樣後序回傳深度，差別在答案是遞迴途中用「左深 + 右深」更新的全域值
- [[0102-Binary-Tree-Level-Order-Traversal]] — 方法二層序骨架的原型題，把「數層數」換成「收集每層的值」
