---
tags:
  - tree
  - pattern
dg-publish: true
---

## 這篇在解決什麼

遞迴的三種 DFS 順序只差**一行的位置**，改起來毫無難度：

```cpp
void dfs(TreeNode* node) {
  if (!node) return;
  // visit(node);   ← 放這裡是前序
  dfs(node->left);
  // visit(node);   ← 放這裡是中序
  dfs(node->right);
  // visit(node);   ← 放這裡是後序
}
```

換成迭代就不是這樣了。**遞迴堆疊幫你記住的不只是「還沒走的節點」，還有「這個節點走到第幾步」**——自己拿 `stack` 模擬時，那個隱含的狀態要另外想辦法表達，於是三種順序變成三種難度：前序最簡單、中序要換一種骨架、後序還得再解決「右子樹回來過了沒」。

這篇收三套專用模板 + 一套統一模板 + Morris，都用 3000 棵隨機樹對照遞迴驗過。

```txt
        1              前序 根左右：1 2 4 5 3 6
      /   \            中序 左根右：4 2 5 1 6 3
     2     3           後序 左右根：4 5 2 6 3 1
    / \   /
   4   5 6             三種順序的差別只在「根」在哪個位置被輸出，
                       左右子樹的相對順序永遠是左先右後
```

## 模板一：前序 — 直接拿 stack 當待辦清單

前序特別簡單，因為**看到節點就能立刻輸出**，不需要記任何狀態。stack 純粹是「還沒處理的節點」清單。

```cpp
// Time: O(n)   每個節點進出 stack 各一次
// Space: O(h)  stack 最多裝一條根到葉的路徑上的兄弟節點
vector<int> preorderTraversal(TreeNode* root) {
  vector<int> out;
  if (!root) {
    return out;
  }
  stack<TreeNode*> st;
  st.push(root);
  while (!st.empty()) {
    TreeNode* node = st.top();
    st.pop();
    out.push_back(node->val);
    if (node->right) {
      st.push(node->right);
    }
    if (node->left) {
      st.push(node->left);
    }
  }
  return out;
}
```

> [!important] 先推右再推左，因為 stack 是後進先出
> 想「先處理左」，就要「後推左」。這個反轉是所有 stack 模擬遞迴的通則，寫錯順序不會出錯只會靜靜地變成「根右左」——反而正是模板三 A 要的東西。

> [!warning] 這個骨架換成 `queue` 不會變成層序的親戚，而是直接**就是**層序
> 同一段程式把 `stack` 換成 `queue`（並改成先左後右）就是 BFS 層序。差別只在「待辦清單的取用順序」，這也是為什麼層序不屬於 DFS 三序、它是另一個維度的走法。

## 模板二：中序 — 一路往左的骨架（最該背熟的一個）

中序不能看到節點就輸出：走到 1 的時候，左子樹還沒走完，1 還不能出場。所以骨架改成**兩段式**——內層 while 一路往左把路徑壓進 stack，彈出時才輸出，然後轉向右子樹。

```cpp
// Time: O(n)
// Space: O(h)
vector<int> inorderTraversal(TreeNode* root) {
  vector<int> out;
  stack<TreeNode*> st;
  TreeNode* cur = root;
  while (cur || !st.empty()) {
    while (cur) {          // 一路往左，把沿路的節點都欠著
      st.push(cur);
      cur = cur->left;
    }
    cur = st.top();        // 左邊走到底了，這個節點可以輸出
    st.pop();
    out.push_back(cur->val);
    cur = cur->right;      // 轉向右子樹，下一輪繼續往左鑽
  }
  return out;
}
```

> [!important] 迴圈條件是 `cur || !st.empty()`，兩個都要
> 只寫 `!st.empty()`：一開始 stack 是空的，迴圈根本不會進去。只寫 `cur`：彈出節點後 `cur = cur->right` 可能是 `nullptr`，但 stack 裡還有欠著的祖先沒輸出，會提早結束。**`cur` 代表「還有新路要鑽」，`st` 代表「還有欠的節點要還」**，任一個非空就沒走完。

> [!tip] 這個骨架是所有 BST 題的地基
> BST 的中序必定嚴格遞增，於是很多題目就是「這個迴圈 + 在 pop 之後做一件事」：[[0098-Validate-Binary-Search-Tree]] 檢查有沒有逆序、[[0230-Kth-Smallest-Element-in-a-BST]] 數到第 k 個就 return、[[0173-Binary-Search-Tree-Iterator]] 更直接——把這個迴圈拆成 `next()` 呼叫之間的狀態，那個 class 的 member 就是 `st` 和 `cur`。

### 前序也能套同一骨架

值得知道但不必特地改用：把 `push_back` 從彈出後搬到壓入時，同一個骨架就變成前序。這正好對應遞迴那三個 `visit()` 位置。

```cpp
while (cur || !st.empty()) {
  while (cur) {
    out.push_back(cur->val);   // ← 只有這一行搬了位置
    st.push(cur);
    cur = cur->left;
  }
  cur = st.top();
  st.pop();
  cur = cur->right;
}
```

## 模板三：後序 — 兩種寫法，難度差很多

後序最麻煩：一個節點被彈出時，你不知道它的右子樹是「還沒走」還是「剛走完回來」。兩種解法各自繞開這個問題。

### 寫法 A：走成「根右左」再整個反轉 — 好寫但要多一次 reverse

`根左右` 反過來是 `右左根`，而 `左右根`（後序）反過來是 `根右左`。所以把模板一的 push 順序對調、得到「根右左」，最後 `reverse` 一次就是後序。

```cpp
// Time: O(n)   多一次 O(n) 反轉
// Space: O(h)  不算輸出陣列
vector<int> postorderTraversal(TreeNode* root) {
  vector<int> out;
  if (!root) {
    return out;
  }
  stack<TreeNode*> st;
  st.push(root);
  while (!st.empty()) {
    TreeNode* node = st.top();
    st.pop();
    out.push_back(node->val);
    if (node->left) {         // 和模板一相反：先推左，讓右先被處理
      st.push(node->left);
    }
    if (node->right) {
      st.push(node->right);
    }
  }
  reverse(out.begin(), out.end());
  return out;
}
```

> [!warning] 它產生的不是「後序過程」，只是「後序結果」
> 節點被實際造訪的順序是根右左——**根最先**。所以凡是需要「處理某節點時子樹的資訊已備妥」的題目（求高度、判平衡、子樹和、釋放記憶體），這個寫法幫不上忙，反轉只救得了輸出順序。要真後序的行為就用寫法 B，或者老實用遞迴。

### 寫法 B：真後序，用 `last` 記住剛走完的子樹

沿用模板二的骨架，多一個 `last` 指標。`peek` 的右子樹存在且不是剛回來的那棵，就先去走右邊；否則代表左右都完成了，可以輸出自己。

```cpp
// Time: O(n)
// Space: O(h)
vector<int> postorderTraversal(TreeNode* root) {
  vector<int> out;
  stack<TreeNode*> st;
  TreeNode *cur = root, *last = nullptr;
  while (cur || !st.empty()) {
    while (cur) {
      st.push(cur);
      cur = cur->left;
    }
    TreeNode* peek = st.top();
    if (peek->right && peek->right != last) {
      cur = peek->right;      // 右邊還沒走，去走右邊（先不 pop）
    } else {
      out.push_back(peek->val);
      last = peek;            // 記住「我這棵子樹走完了」
      st.pop();
    }
  }
  return out;
}
```

> [!important] `last` 就是遞迴幫你記住的「走到第幾步」
> 沒有它，`peek` 每次被看到都會再往右鑽一次 → 無窮迴圈。這是三種順序中唯一需要額外狀態的一種，也是「迭代後序比較難」的全部原因。注意條件成立時**不能 pop**，因為走完右子樹還要回來輸出這個節點。

## 統一模板：NULL 標記法 — 一套骨架三種順序

如果不想背三套，可以只背這一套：**節點被推兩次**，第二次推的後面緊跟一個 `nullptr` 當標記。彈出時看到 `nullptr` 就代表「下一個才是該輸出的」，否則就是「展開這個節點」。改順序只要改 push 的排列。

```cpp
// Time: O(n)   每個節點被 push 兩次
// Space: O(h)  但常數 ~3 倍
vector<int> traversal(TreeNode* root) {
  vector<int> out;
  stack<TreeNode*> st;
  if (root) {
    st.push(root);
  }
  while (!st.empty()) {
    TreeNode* node = st.top();
    st.pop();
    if (node) {
      // 逆序推入：想先處理的最後推。以下是中序（左根右）
      if (node->right) {
        st.push(node->right);
      }
      st.push(node);
      st.push(nullptr);       // 標記：node 自己已就緒，還沒輸出
      if (node->left) {
        st.push(node->left);
      }
    } else {
      out.push_back(st.top()->val);
      st.pop();
    }
  }
  return out;
}
```

三種順序只差 push 的排列（都記得**逆著寫**）：

| 順序 | push 排列（由上到下讀就是逆序）           |
| ---- | ----------------------------------------- |
| 前序 | `right` → `left` → `node, nullptr`         |
| 中序 | `right` → `node, nullptr` → `left`         |
| 後序 | `node, nullptr` → `right` → `left`         |

> [!note] 代價是 stack 峰值約 3 倍，時間同數量級
> 實測（`-O2`，中序，同一棵樹跑 5 次取平均）：
>
> ```txt
> 滿二元樹 n = 1048575        時間        stack 峰值
>   中序專用模板              3.4-3.8 ms      20
>   統一模板（NULL 標記）      3.9-4.8 ms      59
>
> 隨機樹   n = 1000000        時間        stack 峰值
>   中序專用模板              7.5-10.6 ms     30
>   統一模板（NULL 標記）      9.5-12.5 ms     90
> ```
>
> 峰值 ≈ 3h 是必然的（每層留下 `node` + `nullptr` + 兄弟）；時間慢個 10–25%，落在同一個數量級、不同機器跑會有出入，別把它當成「明顯比較慢」。**這是一個用常數換記憶量的交易**，面試白板上很划算。

## Morris：O(1) 空間，代價是暫時改動樹

不用 stack，改成把「左子樹最右節點」的空 `right` 指標暫時接回目前節點當作回程線索，走過後再拆掉。

```cpp
// Time: O(n)   每條邊最多走兩次
// Space: O(1)  不含輸出
vector<int> inorderTraversal(TreeNode* root) {
  vector<int> out;
  TreeNode* cur = root;
  while (cur) {
    if (!cur->left) {
      out.push_back(cur->val);
      cur = cur->right;
    } else {
      TreeNode* pred = cur->left;                    // 找中序前驅
      while (pred->right && pred->right != cur) {
        pred = pred->right;
      }
      if (!pred->right) {
        pred->right = cur;                           // 架線
        cur = cur->left;
      } else {
        pred->right = nullptr;                       // 拆線（第二次來到）
        out.push_back(cur->val);
        cur = cur->right;
      }
    }
  }
  return out;
}
```

把 `out.push_back(cur->val)` 從「拆線那支」搬到「架線那支」（`if (!pred->right)` 裡面）就是 Morris 前序。後序也做得到，但要反轉右邊界鏈、程式碼長度翻倍，實務上不值得。

> [!warning] 遍歷「途中」樹是壞的
> 架線期間樹暫時不是一棵樹（有環），只有走完才復原。所以不能中途 `return`（線會留著、樹永久壞掉），多執行緒或「不允許修改輸入」的場合也不能用。
>
> 而且它**不會比較快**：同上實測，滿二元樹 5.7 ms、隨機樹 13.1 ms，比 stack 版慢約 1.7 倍——因為每個有左子樹的節點都要多走一趟去找前驅。它換到的東西只有空間。

## 怎麼選

| 情況                                     | 用什麼                                   |
| ---------------------------------------- | ---------------------------------------- |
| 沒有特別要求                             | **遞迴**，最短最清楚，實測也最快          |
| BST 題、需要「上一個中序節點」            | 模板二（或它的 class 化 → 0173）          |
| 面試官明講「不能用遞迴」                  | 前序用模板一、中序用模板二、後序用寫法 B  |
| 只想背一套                               | 統一模板                                 |
| 樹極斜、怕爆堆疊（h ~ n）                 | 任一迭代版；或 Morris                     |
| 面試官明講 O(1) 空間                      | Morris                                   |
| 需要「處理節點時子樹資訊已備妥」          | 遞迴後序或寫法 B，**不能**用寫法 A        |

> [!tip] 迭代真正的用途不是取代遞迴，是拆開遞迴
> 大多數樹題遞迴就是最佳解。迭代的價值在於它把「走到哪裡」變成了**可以存起來的顯式狀態**——這是 [[0173-Binary-Search-Tree-Iterator]] 那種「每次呼叫吐一個值」的題目非用不可的原因，遞迴沒辦法在函式中間暫停。

## 題目分類

| 模板                | 題目                                                                                                   |
| ------------------- | ------------------------------------------------------------------------------------------------------ |
| 模板一（前序）      | [[0144-Binary-Tree-Preorder-Traversal]]、[[0100-Same-Tree]]、[[0112-Path-Sum]]                          |
| 模板二（中序）      | [[0094-Binary-Tree-Inorder-Traversal]]、[[0098-Validate-Binary-Search-Tree]]、[[0230-Kth-Smallest-Element-in-a-BST]]、[[0173-Binary-Search-Tree-Iterator]] |
| 模板三（後序）      | [[0145-Binary-Tree-Postorder-Traversal]]、[[0104-Maximum-Depth-of-Binary-Tree]]、[[0110-Balanced-Binary-Tree]]、[[0543-Diameter-of-Binary-Tree]] |
| 層序（`queue`）     | [[0102-Binary-Tree-Level-Order-Traversal]]、[[0199-Binary-Tree-Right-Side-View]]、[[0111-Minimum-Depth-of-Binary-Tree]] |

後序那一列的三題在 LeetCode 上都寫成遞迴，列在這裡是因為它們的**資訊流動方向**是後序——子樹先回報、父節點才能算——正是寫法 A 幫不上忙、寫法 B 才能改寫的那一類。

## Related Problems

[[0094-Binary-Tree-Inorder-Traversal]] — 模板二最乾淨的原型
[[0144-Binary-Tree-Preorder-Traversal]] — 模板一最乾淨的原型
[[0145-Binary-Tree-Postorder-Traversal]] — 模板三兩種寫法的對照場
[[0098-Validate-Binary-Search-Tree]] — 模板二 + 一個 `prev` 指標的代表題
[[0230-Kth-Smallest-Element-in-a-BST]] — 模板二 + 提早 return 的代表題
[[0173-Binary-Search-Tree-Iterator]] — 把模板二拆成 class 狀態，迭代非用不可的場合
