---
leetcode-id: 230
difficulty: medium
tags:
  - tree
  - depth-first-search
  - binary-search-tree
  - grind-169
  - neetcode-150
memo: kth smallest 就是中序走到第 k 個就停，所以是 O(h + k) 而不是 O(n)；follow up 問「改得頻繁又查得頻繁」，答案是在節點上增廣子樹 size 讓排名查詢變成一路往下走，pbds 的 order statistic tree 就是紅黑樹加這個增廣、不是另一種資料結構
dg-publish: true
---

## Problem Description

Given the `root` of a binary search tree, and an integer `k`, return the `k`<sup>th</sup> smallest value (**1-indexed**) of all the values of the nodes in the tree.

**Follow up:** If the BST is modified often (i.e., we can do insert and delete operations) and you need to find the kth smallest frequently, how would you optimize?

## Solution

核心觀念：**BST 的中序走訪就是由小到大的排序結果**，所以「第 k 小」＝「中序的第 k 個」。這件事一講就完，真正的重點在後半段——`k` 通常遠小於 `n`，所以**走到第 k 個就該停**，不必走完整棵樹；而 follow up 問的是完全不同的問題：當樹一直在變、查詢又很頻繁時，連「走 k 步」都嫌慢，那就要把**排名資訊存進節點裡**。

```txt
        5              中序：1 2 3 4 5 6
      /   \            k=3 → 走到 3 就 return，右半邊完全沒碰
     3     6           k=1 → 只往左鑽到底，走 h 步
    / \
   2   4               所以複雜度是 O(h + k)：先鑽到最左（h 步），再走 k 步
  /
 1
```

### 方法一：中序迭代 + 提早 return — O(h + k)／O(h)（推薦）

```cpp
// Time: O(h + k)  先鑽到最左端 h 步，之後每彈出一個就消耗一個 k
// Space: O(h)     stack 最多裝一條根到葉的路徑
class Solution {
 public:
  int kthSmallest(TreeNode* root, int k) {
    stack<TreeNode*> st;
    TreeNode* cur = root;
    while (cur || !st.empty()) {
      while (cur) {
        st.push(cur);
        cur = cur->left;
      }
      cur = st.top();
      st.pop();
      if (--k == 0) {
        return cur->val;
      }
      cur = cur->right;
    }
    return -1;  // 題目保證 1 <= k <= n，走不到這裡
  }
};
```

> [!important] 迭代在這題不只是「不用遞迴」，它讓提早結束變得理所當然
> 遞迴要停在中間得多帶一個旗標層層 return，迭代版只是一句 `return`。骨架本身（`cur` 一路往左壓進 stack、彈出時才處理）整理在 [[Tree-Traversal-Iterative]]，是所有 BST 題共用的地基。

> [!tip] 複雜度是 O(h + k)，不是 O(n) 也不是 O(k)
> `h` 是鑽到最左端的成本、躲不掉；`k` 是之後每次彈出消耗一個。`k = 1` 時就是 O(h)。面試講 O(n) 會被追問——雖然最壞 `k = n` 確實退化成 O(n)。

### 方法二：中序遞迴 + 計數剪枝 — O(h + k)／O(h)

想用遞迴的話，關鍵是**找到之後要能一路 return**，否則還是走完全樹：

```cpp
// Time: O(h + k)
// Space: O(h)
class Solution {
  int k, ans = -1;

  void dfs(TreeNode* node) {
    if (!node || k == 0) {
      return;
    }
    dfs(node->left);
    if (k == 0) {  // 左子樹裡已經找到了，別再往下
      return;
    }
    if (--k == 0) {
      ans = node->val;
      return;
    }
    dfs(node->right);
  }

 public:
  int kthSmallest(TreeNode* root, int k) {
    this->k = k;
    dfs(root);
    return ans;
  }
};
```

> [!warning] 少了那兩個 `k == 0` 的檢查，複雜度就悄悄變回 O(n)
> 少寫也還是會給出正確答案——`--k` 之後 `k` 變負，不會再有 `--k == 0` 成立，`ans` 不會被覆蓋。**它只是白走完剩下的所有節點**。這種「答案對但複雜度錯」的 bug 測不出來，只能靠讀。

> [!note] Morris 中序在這題不划算
> 它能做到 O(1) 空間，但**不能中途 `return`**——線還沒拆完，樹會永久留著環。所以只能走完全程、只是不再更新答案，等於放棄了這題最重要的提早結束。除非面試官明講「O(1) 空間」，否則別用。

### 方法三：在節點上增廣子樹大小 — 查詢 O(h)／更新 O(h)（Follow up 的答案）

Follow up 的關鍵洞見：**「第 k 小」是一個排名查詢，而排名可以從子樹大小直接算出來**。每個節點多存一個 `size`（我這棵子樹有幾個節點），查詢就變成一路往下走，不走訪任何多餘節點：

```cpp
// Time: 查詢 O(h)／insert、erase 各 O(h)
// Space: O(n) 存 size 欄位（每節點 O(1)）
struct Node {
  int val, size;  // size = 這棵子樹的節點數
  Node *left, *right;
  Node(int v) : val(v), size(1), left(nullptr), right(nullptr) {}
};

int sz(Node* n) { return n ? n->size : 0; }
void pull(Node* n) { n->size = 1 + sz(n->left) + sz(n->right); }

int kth(Node* node, int k) {  // 1-indexed
  while (node) {
    int l = sz(node->left);
    if (k == l + 1) {
      return node->val;  // 左邊剛好 k-1 個，就是我
    }
    if (k <= l) {
      node = node->left;  // 答案在左子樹，k 不變
    } else {
      k -= l + 1;  // 去右子樹，扣掉左子樹 + 自己
      node = node->right;
    }
  }
  return -1;
}

Node* insert(Node* node, int v) {
  if (!node) {
    return new Node(v);
  }
  if (v < node->val) {
    node->left = insert(node->left, v);
  } else if (v > node->val) {
    node->right = insert(node->right, v);
  } else {
    return node;  // 已存在，size 不變
  }
  pull(node);  // 回程順手更新
  return node;
}

Node* erase(Node* node, int v) {
  if (!node) {
    return nullptr;
  }
  if (v < node->val) {
    node->left = erase(node->left, v);
  } else if (v > node->val) {
    node->right = erase(node->right, v);
  } else {
    if (!node->left) {
      Node* r = node->right;
      delete node;
      return r;
    }
    if (!node->right) {
      Node* l = node->left;
      delete node;
      return l;
    }
    Node* succ = node->right;  // 右子樹最小者頂上來
    while (succ->left) {
      succ = succ->left;
    }
    node->val = succ->val;
    node->right = erase(node->right, succ->val);
  }
  pull(node);
  return node;
}
```

> [!important] 維護 `size` 是免費的，因為只有「根到該節點」這條路徑會變
> 插入或刪除只影響祖先鏈上每個節點的計數，而那條鏈**剛好就是遞迴回來的路**——在回程補一句 `pull(node)` 就結束了，不增加任何複雜度。這是增廣資料結構的通則：**只要那個附加資訊能從左右子樹 O(1) 合併出來，就能沿著回程順手維護。**

> [!tip] 反向操作 `rank(v)` 白送給你
> 同一套 `size` 也能回答「有幾個元素小於 v」——往右走時把「左子樹 + 自己」的數量累加起來即可。`kth` 和 `rank` 這一對合起來就是 order statistic tree 的完整介面。
>
> ```cpp
> int rankOf(Node* node, int v) {
>   int r = 0;
>   while (node) {
>     if (v <= node->val) {
>       node = node->left;
>     } else {
>       r += sz(node->left) + 1;
>       node = node->right;
>     }
>   }
>   return r;
> }
> ```

> [!note] pbds 的 order statistic tree 就是這一招，不是另一種資料結構
> `__gnu_pbds` 的 `tree<int, null_type, less<int>, rb_tree_tag, tree_order_statistics_node_update>`，policy 名字已經說白了：它是**紅黑樹 + 每個節點存子樹 size**。`find_by_order(k)` 是上面的 `kth`、`order_of_key(v)` 是上面的 `rankOf`，走法一模一樣。實測（先建 10 萬節點，再做 2 萬次隨機 k 查詢）：
>
> ```txt
>   中序走訪 O(k)       36062.4 ms
>   手刻增廣 size O(h)     10.2 ms   (3523x)
>   pbds find_by_order      8.9 ms
> ```
>
> 手刻版跟它同速。所以**面試 follow up 要聽的是「在節點增廣 size」，不是「換一個容器」**——講得出 `size` 怎麼維護，比講得出 pbds 的名字有價值。

> [!warning] 增廣解決的是「走 k 步」，沒有解決「樹很高」
> `kth` 從 O(k) 變成 O(h)，但 `h` 本身可能很糟。手刻 BST 照順序插入就退化成一條鏈：
>
> ```txt
> n=20000，2 萬次 kth 查詢
>   有序插入   樹高 20000    364.22 ms
>   打亂插入   樹高    33      2.94 ms
> ```
>
> 所以完整答案是**兩件事疊起來**：平衡（AVL／紅黑／Treap）保證 `h = O(log n)`，加上 size 增廣把排名變成沿路計算——合起來才叫 order statistic tree，insert／erase／kth 全部 O(log n)。只講其中一半都不算答完。

> [!note] 不能改 `TreeNode` 結構時的備案
> 外掛一張 `unordered_map<TreeNode*, int> size`，查詢一樣 O(h)，代價是每次改動要沿路更新這張表、常數大得多。面試裡可以當作「如果不允許動 struct，我會這樣」的補充。

## Related Problems

- [[0098-Validate-Binary-Search-Tree]] — 同一個中序骨架，把「數到第 k 個」換成「檢查有沒有逆序」
- [[Tree-Traversal-Iterative]] — 方法一與方法二共用的中序模板，含前序／後序的對照
- [[0173-Binary-Search-Tree-Iterator]] — 把方法一的迴圈拆成 class 狀態，`next()` 就是「每呼叫一次前進一格」
- [[0703-Kth-Largest-Element-in-a-Stream]] — 同樣是「動態資料上的第 k 名」，但那題用固定大小的 heap，正好是本題方法三的對照組
- [[0450-Delete-Node-in-a-BST]] — 方法三 `erase` 裡「用後繼頂上」的那段，是那題的完整主題
