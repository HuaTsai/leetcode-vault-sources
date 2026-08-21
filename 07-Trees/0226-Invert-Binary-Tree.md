---
leetcode-id: 226
difficulty: easy
tags:
  - tree
  - depth-first-search
  - breadth-first-search
  - grind-169
  - neetcode-150
memo: 反轉的本質是對每個節點交換左右「指標」，整棵子樹跟著搬家，所以交換與往下走互不干擾、前序後序層序都對；陷阱是先把 left 覆寫掉再拿 left 去遞迴，第二次讀到的已經是新指標，右子樹會被複製兩份、左子樹整個消失
dg-publish: true
---

## Problem Description

Given the `root` of a binary tree, invert the tree, and return *its root*.

## Solution

核心觀念：反轉整棵樹 = **對每一個節點做同一件事——交換它的左右孩子**。關鍵在於交換的是「指標」不是「值」，`swap(left, right)` 一行就讓兩棵子樹整個換邊，子樹內部完全不用理會。因為節點自己的交換不依賴子樹是否已經處理完，走訪順序完全自由：前序、後序、層序、遞迴、迭代都對。這也是這題常被當成「二元樹走訪模板」入門題的原因。

```txt
        4                          4
      /   \                      /   \
     2     7         →          7     2
    / \   / \                  / \   / \
   1   3 6   9                9   6 3   1

節點 4：把 left/right 兩個指標對調 → 2、7 兩棵子樹整個換邊
節點 7（現在在左邊）：再對調一次 → 6、9 換邊
每個節點只被處理一次，處理內容都一樣
```

### 方法一：遞迴 DFS — O(n)／O(h)（推薦）

```cpp
// Time: O(n)   每個節點恰好造訪一次，每次 O(1)
// Space: O(h)  遞迴堆疊深度＝樹高，平衡樹 O(log n)、退化成鏈 O(n)
class Solution {
 public:
  TreeNode *invertTree(TreeNode *root) {
    if (!root) {
      return nullptr;
    }
    swap(root->left, root->right);
    invertTree(root->left);
    invertTree(root->right);
    return root;
  }
};
```

> [!important] 交換的是指標，整棵子樹跟著走
> `swap(root->left, root->right)` 動到的只有 `root` 裡的兩個指標欄位，但被指到的兩棵子樹連同它們的所有後代都一起換到另一邊。所以「交換這一層」與「反轉子樹」是兩件獨立的事，各做各的，合起來就是整棵樹的鏡像。若誤以為要搬移節點內容而去交換 `val`，那只是換了根的數字、結構沒動，答案是錯的。

> [!warning] 別先覆寫 `left` 再拿 `left` 去遞迴
> 常見寫法 `root->left = invertTree(root->right); root->right = invertTree(root->left);` 看起來對稱，其實第二行讀到的 `root->left` 已經是第一行剛寫進去的**新**指標（原本的右子樹），於是右子樹被反轉兩次並複製到兩邊，原本的左子樹整個遺失。實測 `[4,2,7,1,3,6,9]` 會得到 `[4,7,7,9,9,9,9]`。要嘛用 `swap` 一次搞定，要嘛先存 `TreeNode *l = root->left;` 再寫回。

> [!tip] 前序或後序都可以，因為兩者不互相依賴
> 把 `swap` 移到兩行遞迴之後（後序）結果一模一樣：節點的交換不看子樹狀態，子樹的反轉也不看父節點。這點跟很多樹題不同——像求高度、判平衡那類必須「先有子樹答案才能算自己」，順序就不能亂調。

### 方法二：BFS 層序迭代 — O(n)／O(n)

換成顯式佇列，把遞迴堆疊改成自己管理的容器，邏輯完全一樣：取出一個節點、交換它的左右、把非空的孩子排進佇列。

```cpp
// Time: O(n)   每個節點進出佇列各一次
// Space: O(n)  佇列最多裝下一整層，滿二元樹最後一層約 n/2 個節點
class Solution {
 public:
  TreeNode *invertTree(TreeNode *root) {
    if (!root) {
      return nullptr;
    }
    queue<TreeNode *> q;
    q.push(root);
    while (!q.empty()) {
      auto fn = q.front();
      q.pop();
      swap(fn->left, fn->right);
      if (fn->left) {
        q.push(fn->left);
      }
      if (fn->right) {
        q.push(fn->right);
      }
    }
    return root;
  }
};
```

> [!tip] `queue` 直接換成 `stack` 就是迭代版 DFS
> 這份程式碼裡「容器的取出順序」是唯一決定走訪次序的東西，而本題對次序無所謂，所以 `queue` → `stack`（`q.front()` → `st.top()`）照樣正確。想避開遞迴又不想吃一整層的空間時可以這樣寫。

> [!note] 這題唯一的取捨是空間，方向還會反過來
> 遞迴是 `O(h)`、BFS 是 `O(n)`，一般情況遞迴省。但樹極度傾斜時（`h = n`，例如題目沒保證平衡的資料）遞迴有爆堆疊風險，而 BFS 佇列裡每層只有 1 個節點、實際只用 `O(1)`。兩者的最壞情況剛好落在相反的樹形上。

## Related Problems

- [[0101-Symmetric-Tree]] — 判斷樹是否等於自己的鏡像，等價於「invert 之後跟原樹相同」，同一個鏡像概念換個問法
- [[0100-Same-Tree]] — 一模一樣的「處理當前節點 + 遞迴左右」骨架，只是把 swap 換成比較
- [[0104-Maximum-Depth-of-Binary-Tree]] — 同樣 DFS／BFS 雙解模板，但它必須先有子樹答案才能算自己，順序不像本題可以亂調
- [[0102-Binary-Tree-Level-Order-Traversal]] — 方法二 BFS 骨架的原型題，多一層「一次處理完整層」的迴圈
- [[0572-Subtree-of-Another-Tree]] — 以 Same Tree 當子程序的樹走訪，練習遞迴的組合
