---
leetcode-id: 235
difficulty: medium
tags:
  - tree
  - depth-first-search
  - binary-search-tree
  - binary-tree
  - lowest-common-ancestor
  - grind-169
  - neetcode-150
memo: 沿樹往下走，第一個讓 p 與 q 分岔（值夾在兩者之間）的節點就是 LCA；祖先關係不必特判，cur 撞到 p 或 q 時兩側條件會同時不成立而自然停下
dg-publish: true
---

## Problem Description

Given a binary search tree (BST), find the lowest common ancestor (LCA) node of two given nodes in the BST.

According to the definition of LCA on Wikipedia: "The lowest common ancestor is defined between two nodes `p` and `q` as the lowest node in `T` that has both `p` and `q` as descendants (where we allow a node to be a descendant of itself)."

Constraints:

- All `Node.val` are unique.
- `p != q`
- `p` and `q` will exist in the BST.

## Solution

BST 的排序性質讓這題不需要「搜尋」。從 root 出發，`p`、`q` 相對於當前節點 `cur` 只有三種關係：

```txt
              cur
             /   \
        兩個都小   兩個都大    → 答案必在那一側，往下走，cur 不可能是 LCA

        一左一右，或 cur 就是 p / q 之一
                             → cur 已經是「同時是兩者祖先」的最低節點，停
```

換句話說，**LCA 就是從根往下走遇到的第一個分岔點**。走的過程中不必回頭、不必記路徑，所以是 O(h) 時間、O(1) 空間。

```txt
        6
      /   \
     2     8         p=2, q=8：6 的左右各一個 → 分岔 → LCA = 6
    / \   / \
   0   4 7   9       p=2, q=4：兩個都 < 6 → 往左到 2
      / \            2 本身就是 p → 停 → LCA = 2
     3   5
```

> [!important] 「第一個分岔點」等價於「第一個值夾在 p、q 之間的節點」
> 假設 `p->val < q->val`，LCA 就是根到底路徑上**第一個滿足 `p->val <= cur->val <= q->val` 的節點**。方法一是前一種說法的直譯，方法二是後一種的直譯，兩者跑的是同一條路徑、同樣的步數，差別只在怎麼把條件寫出來。

> [!tip] 「一個是另一個的祖先」不需要特判
> 當 `cur == p` 時，`p->val < cur->val` 與 `p->val > cur->val` **同時為假**，兩個 `if` 都不成立，自然落到 `return cur`。很多人會多寫一段 `if (cur == p || cur == q) return cur;`——不是錯，是多餘。這也是本題面試最愛追問的一點。

### 方法一：往下走到第一個分岔點 — O(h)／O(1)

直接把上面三種關係寫成三個分支。`h` 是樹高，平衡時 O(log n)，退化成鏈時 O(n)。

```cpp
// Time: O(h)   只走一條根到 LCA 的路徑，不回頭
// Space: O(1)  迭代，沒有遞迴堆疊也沒有額外容器
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
  TreeNode* cur = root;
  while (cur) {
    if (p->val < cur->val && q->val < cur->val) {
      cur = cur->left;
    } else if (p->val > cur->val && q->val > cur->val) {
      cur = cur->right;
    } else {
      return cur;  // 分岔點，或 cur 本身就是 p / q
    }
  }
  return nullptr;  // 題目保證 p、q 都在樹中，走不到這裡
}
```

### 方法二：先 normalize，找第一個落在區間內的節點 — O(h)／O(1)

先 `swap` 讓 `p->val < q->val`，兩個節點就變成一個區間 `[p->val, q->val]`。之後每層只要問「`cur` 在區間的哪一邊」，比較次數從 4 次降到 2 次，`else` 分支的語意也直接寫在 code 上。

```cpp
// Time: O(h)
// Space: O(1)
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
  if (p->val > q->val) {
    swap(p, q);  // 之後只需假設 p->val < q->val
  }
  for (TreeNode* cur = root;;) {
    if (cur->val > q->val) {
      cur = cur->left;   // 整個區間都在左子樹
    } else if (cur->val < p->val) {
      cur = cur->right;  // 整個區間都在右子樹
    } else {
      return cur;        // p->val <= cur->val <= q->val
    }
  }
}
```

> [!warning] 用乘積判同側的「聰明寫法」會整數溢位
> 網路上常見這種一行迴圈：
>
> ```cpp
> while ((p->val - cur->val) * (q->val - cur->val) > 0) {  // 會溢位
>   cur = p->val < cur->val ? cur->left : cur->right;
> }
> ```
>
> `Node.val` 範圍是 ±10⁹，**光是 `p->val - cur->val` 這個差值就超出 int**。實測 `root=0, p=-1e9, q=1e9`：正確乘積是 -10¹⁸，int 算出 `1486618624`（正數），於是誤判成同側往左走，回傳 -1e9 而非 0。真要這樣寫，兩個差值都得先轉 `long long`。

### 方法三：遞迴版 — O(h)／O(h)

同一套判斷寫成遞迴，讀起來最貼近「往哪一側走」的直覺，代價是多了 O(h) 的堆疊。三個分支剛好對應樹的三種關係，沒有 `nullptr` 檢查是因為題目保證 `p`、`q` 存在，遞迴一定會在走到空之前停在分岔點。

```cpp
// Time: O(h)
// Space: O(h)  遞迴堆疊；這是尾遞迴，開了最佳化通常會被編譯器攤平成迴圈
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
  if (p->val < root->val && q->val < root->val) {
    return lowestCommonAncestor(root->left, p, q);
  }
  if (p->val > root->val && q->val > root->val) {
    return lowestCommonAncestor(root->right, p, q);
  }
  return root;
}
```

> [!note] 沒有 BST 性質，或同一棵樹要問很多次
> 拿掉 BST 性質後這題就是 [[0236-Lowest-Common-Ancestor-of-a-Binary-Tree]]，只能靠後序遞迴讓左右子樹各自回報「有沒有找到 p 或 q」，退化成 O(n)。若同一棵樹要回答**很多次** LCA query，才輪到倍增祖先表之類的預處理技巧，見 [[Binary-Lifting-LCA]]。

## Related Problems

- [[0236-Lowest-Common-Ancestor-of-a-Binary-Tree]] — 同一題拿掉 BST 條件，只能後序遞迴、O(n)
- [[1650-Lowest-Common-Ancestor-of-a-Binary-Tree-III]] — 節點帶 parent 指標，退化成兩條鏈求交點
- [[0098-Validate-Binary-Search-Tree]] — 同樣用「值必須落在某個區間內」的觀點看 BST
- [[0230-Kth-Smallest-Element-in-a-BST]] — 一樣靠排序性質一路剪掉半棵子樹
- [[Binary-Lifting-LCA]] — 多次 query 時的倍增祖先表解法
