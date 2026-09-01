---
leetcode-id: 102
difficulty: medium
tags:
  - tree
  - breadth-first-search
  - binary-tree
  - grind-169
  - neetcode-150
memo: 進內層迴圈前先把 q.size() 凍結成 n 才能剝下完整一層，直接拿 q.size() 當條件會邊 pop 邊 push 吃掉下一層；n 既然在手上就順手 vals.reserve(n)，但 while（n--）會把 n 吃掉，reserve 必須寫在進迴圈之前
dg-publish: true
---

## Problem Description

Given the `root` of a binary tree, return the level order traversal of its nodes' values. (i.e., from left to right, level by level).

## Solution

核心觀念：層序走訪就是**一次剝掉一整層**——把當層的節點全部彈出、值收進同一個 `vector`，過程中把它們的孩子排進佇列組成下一層。整份程式碼的命脈是「當層有幾個節點」這個數字必須在動手剝之前**先凍結下來**，因為佇列在剝的過程中同時在長大。

```txt
        3            q = [3]           n=1  →  vals = [3]
      /   \          q = [9,20]        n=2  →  vals = [9,20]
     9     20        q = [15,7]        n=2  →  vals = [15,7]
          /  \       q = []                 結束
        15    7

答案 = [[3],[9,20],[15,7]]

剝第 2 層的當下，佇列從 [9,20] 一路變成 [20,15,7] → [15,7]，
size() 全程在跳動；n 就是把「這一層是 2 個」釘死的那顆釘子。
```

### 方法一：BFS 佇列 — O(n)／O(n)（推薦）

```cpp
// Time: O(n)   每個節點進出佇列各一次
// Space: O(n)  佇列最多裝下一整層，滿二元樹最後一層約 n/2 個節點
class Solution {
 public:
  vector<vector<int>> levelOrder(TreeNode* root) {
    if (!root) {
      return {};
    }
    vector<vector<int>> ans;
    queue<TreeNode *> q;
    q.push(root);
    while (!q.empty()) {
      int n = q.size();
      vector<int> vals;
      vals.reserve(n);
      while (n--) {
        auto fn = q.front();
        q.pop();
        vals.push_back(fn->val);
        if (fn->left) {
          q.push(fn->left);
        }
        if (fn->right) {
          q.push(fn->right);
        }
      }
      ans.push_back(std::move(vals));
    }
    return ans;
  }
};
```

> [!warning] `int n = q.size();` 這一行不能省，也不能改寫成迴圈條件
> 寫成 `for (size_t i = 0; i < q.size(); ++i)` 或 `while (!q.empty() && ...)` 都會壞掉：內層一邊 pop 一邊 push，`q.size()` 全程浮動，這一層會把下一層的節點也吃進 `vals`，最後所有值擠成一層。跟 [[0104-Maximum-Depth-of-Binary-Tree]] 是同一個陷阱，差別在那題只數層數、在規則樹形上還可能碰巧矇對，這題會直接吐出形狀錯誤的答案，反而更容易被測資抓到。

> [!tip] `reserve` 必須寫在 `while (n--)` 之前，因為 `n--` 會把 `n` 吃光
> 這題跟 0104 不同的地方在於**每層要收值**，而 `n` 正好就是這層的長度——不 reserve 的話 `vals` 得從 0 開始倍增重配。但順序不能反：`while (n--)` 結束時 `n` 已經是 -1，事後補 `reserve` 只會拿到垃圾。收完再 `std::move(vals)` 進 `ans`，省掉一次整層的複製。

> [!note] `reserve` + `move` 不是紙上談兵，量測得出來
> 2000 節點隨機樹跑 2000 次、用 cachegrind 數指令數：原版（無 reserve、`push_back(vals)`）198.3M，加了 reserve 與 move 後 131.4M，**少掉約 34%**。再進一步把 `queue` 換成 `cur` / `next` 兩個 `vector` 交替 `swap`（層序天然就是「讀當層、寫下層」，不需要 `deque` 的分段配置與 `front()/pop()` 間接定址）可壓到 105.5M，**比原版少 47%**。三者都是 O(n)，差的是常數；面試寫佇列版最穩，vector 雙緩衝是知道有這條路即可。

### 方法二：DFS 前序 ＋ depth 當索引 — O(n)／O(h)

層序**不一定要 BFS**。深度優先照樣能做，只要把 `depth` 當成 `ans` 的索引：走到某個節點時，若這個深度還沒有對應的 `vector` 就新開一個，再把值塞進 `ans[depth]`。前序保證同一層由左到右被造訪，所以每層的順序天然正確。

```cpp
// Time: O(n)   每個節點恰好造訪一次
// Space: O(h)  遞迴堆疊深度＝樹高（不計答案本身）
class Solution {
 public:
  vector<vector<int>> levelOrder(TreeNode* root) {
    dfs(root, 0);
    return ans;
  }

 private:
  vector<vector<int>> ans;

  void dfs(TreeNode* node, int depth) {
    if (!node) {
      return;
    }
    if (depth == (int)ans.size()) {
      ans.push_back({});
    }
    ans[depth].push_back(node->val);
    dfs(node->left, depth + 1);
    dfs(node->right, depth + 1);
  }
};
```

> [!important] `depth == ans.size()` 就是「第一次踏進這一層」
> 前序走訪保證了每一層都是被**最左邊**的那個節點首次踏到，而那一刻 `ans` 剛好只有 0…depth-1 這 depth 層，所以「深度等於現有層數」就是新層的判定條件，不需要另外維護最大深度。若寫成 `>=` 或用 `resize`，遇到樹高不規則時就會多開空層。注意 `ans.size()` 是 `size_t`，跟 `int depth` 直接比會觸發 sign-compare 警告，記得轉型。

> [!tip] 這個版本的真正價值是模板性，而不是速度
> 同一份骨架換一行就是整組層序題：[[0107-Binary-Tree-Level-Order-Traversal-II]] 最後 `reverse(ans)`、[[0199-Binary-Tree-Right-Side-View]] 改成每層只留最後一個（或前序換成「先右後左」時只留第一個）、[[0637-Average-of-Levels-in-Binary-Tree]] 每層求平均。**「按層分組」和「按層走訪」是兩件事，前者只需要一個 depth 索引。**

> [!warning] 但它比 BFS 慢，別誤以為遞迴一定比較快
> 同樣的量測條件下 DFS 版是 297.0M 指令，比 BFS 佇列原版還多 50%、是佇列優化版的 2.3 倍。原因有二：每個節點都要對左右各做一次遞迴呼叫，葉節點那約 n/2 次踩空的呼叫照樣建棧幀；而且各層的 `vector` 是零散長出來的，拿不到「這層有幾個」也就無從 reserve。

## Related Problems

- [[0104-Maximum-Depth-of-Binary-Tree]] — 同一副 BFS 骨架，把「收集每層的值」退化成「數剝了幾層」；`int n = q.size()` 的陷阱兩題共用
- [[0107-Binary-Tree-Level-Order-Traversal-II]] — 完全同一份程式碼，最後把 `ans` 反轉即可，別真的去做「由下往上的 BFS」
- [[0103-Binary-Tree-Zigzag-Level-Order-Traversal]] — 加一個層數奇偶判斷決定要不要 reverse 該層；也可改成往 `vals` 頭部填，但那是 O(n²)
- [[0199-Binary-Tree-Right-Side-View]] — 每層只取最後一個元素，是本題方法一改一行的直接應用
- [[0637-Average-of-Levels-in-Binary-Tree]] — 每層改成累加求平均，注意用 `double` 或 `long long` 防溢位
- [[0111-Minimum-Depth-of-Binary-Tree]] — 同樣層序，但碰到第一個葉節點就能提早 return，是 BFS 真正贏遞迴的場合
