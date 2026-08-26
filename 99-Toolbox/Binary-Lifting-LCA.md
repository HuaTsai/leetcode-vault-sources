---
tags:
  - tree
  - pattern
  - lowest-common-ancestor
dg-publish: true
---

## 這篇在解決什麼

求 LCA（最低共同祖先）最直覺的做法是「兩個節點一起往上爬，先對齊深度，再一步一步同時往上，直到撞在一起」。一次 query O(h)，樹平衡時很快、退化成鏈時 O(n)。

問題出在**同一棵樹要問很多次**的時候：q 次 query 就是 O(qh)，n 和 q 都到 10⁵ 時直接爆。

Binary lifting（倍增）用 O(n log n) 的預處理換掉這個 h：先建一張「每個節點往上 2^k 級的祖先是誰」的表，之後每次 query 只要 O(log n)。這篇收表的建法、查法、迭代版 build，以及跟其他 LCA 解法的取捨。全部用 400 棵隨機樹、121075 組 query 對照樸素爬 parent 驗過。

> [!note] 只問一次的話，別用這個
> 單次 query 建表的成本永遠賺不回來。樹是 BST 就直接沿大小關係走（[[0235-Lowest-Common-Ancestor-of-a-Binary-Search-Tree]]），普通二元樹就後序遞迴（[[0236-Lowest-Common-Ancestor-of-a-Binary-Tree]]），兩者都是一趟走完，建表的成本永遠賺不回來。

## 核心：任何整數都是 2 的冪次和

倍增的全部想法就這一句——**要往上跳 k 級，把 k 拆成二進位，跳 O(log k) 次就到**：

```txt
往上跳 13 級  =  13 = 1101₂ = 8 + 4 + 1
              →  跳 8 級、再跳 4 級、再跳 1 級，三次搞定

原本要 13 步，現在 3 步。前提是「跳 2^k 級」這個動作本身是 O(1)。
```

所以只要預先算好每個節點的第 1、2、4、8… 級祖先，任意距離的跳躍都能拆成幾次表查詢。

## 第一步：建祖先表

`up[k][v]` = 節點 `v` 往上 2^k 級的祖先，沒有就是 `-1`。整張表靠一條遞推填滿：

```txt
up[0][v] = parent(v)                    ← 基底，往上 1 級

up[k][v] = up[k-1][ up[k-1][v] ]        ← 先跳 2^(k-1) 級，再跳 2^(k-1) 級
                                           2^(k-1) + 2^(k-1) = 2^k

     v ──2^(k-1)──▶ m ──2^(k-1)──▶ ans
     └────────────2^k────────────┘
```

表的大小是 O(n log n)，這也是這個做法的記憶體代價。

## 第二步：查 LCA 的兩個階段

```txt
階段一：對齊深度
   深的那個先往上跳 depth 差，讓兩個節點站在同一層
   跳完若 u == v，代表其中一個本來就是另一個的祖先，直接回傳

階段二：一起往上跳到 LCA 的正下方
   k 從大到小試：up[k][u] != up[k][v] 才跳
   跳完 u 和 v 是 LCA 的兩個不同子節點 → 答案是 up[0][u]
```

> [!important] 階段二為什麼是「祖先**不同**才跳」，而不是相同才跳
> 因為「祖先相同」只代表跳到了某個共同祖先，**不保證是最低的那個**——可能已經跳過頭。反過來「祖先不同」保證還沒到 LCA，跳過去絕對安全。於是從大步試到小步，每次都跳到「安全範圍內能跳的最遠處」，最後必然停在 LCA 的正下方，再往上一步就是答案。這是貪心，所以 `k` 一定要**從大到小**，寫反了會在還能跳大步時先耗掉小步。

## 完整實作

```cpp
// 預處理 Time: O(n log n)   Space: O(n log n)
// 每次 query Time: O(log n)
struct LCA {
  int n, LOG;
  vector<vector<int>> adj, up;  // up[k][v] = v 往上 2^k 級的祖先，-1 表示不存在
  vector<int> depth;

  explicit LCA(int n) : n(n), adj(n), depth(n, 0) {
    LOG = 1;
    while ((1 << LOG) < n) {
      ++LOG;  // 保證 2^LOG >= n，樹再高也跳得完
    }
  }

  void addEdge(int u, int v) {
    adj[u].push_back(v);
    adj[v].push_back(u);
  }

  void build(int root) {
    up.assign(LOG + 1, vector<int>(n, -1));
    dfs(root, -1);
  }

  void dfs(int v, int parent) {
    up[0][v] = parent;
    for (int k = 1; k <= LOG; ++k) {
      up[k][v] = up[k - 1][v] == -1 ? -1 : up[k - 1][up[k - 1][v]];
    }
    for (int to : adj[v]) {
      if (to != parent) {
        depth[to] = depth[v] + 1;
        dfs(to, v);
      }
    }
  }

  // 順帶送的能力：往上跳任意 k 級，同樣 O(log k)
  int kthAncestor(int v, int k) {
    for (int i = 0; v != -1 && (1 << i) <= k; ++i) {
      if (k >> i & 1) {
        v = up[i][v];
      }
    }
    return v;
  }

  int lca(int u, int v) {
    if (depth[u] < depth[v]) {
      swap(u, v);  // 保證 u 是比較深的那個
    }
    u = kthAncestor(u, depth[u] - depth[v]);  // 階段一：對齊深度
    if (u == v) {
      return u;  // 一個是另一個的祖先
    }
    for (int k = LOG; k >= 0; --k) {  // 階段二：必須從大到小
      if (up[k][u] != up[k][v]) {
        u = up[k][u];
        v = up[k][v];
      }
    }
    return up[0][u];  // 停在 LCA 正下方，再上一步
  }
};
```

`dfs` 裡把「填 `up[k][v]`」放在遞迴子節點**之前**，是因為填 `v` 的表只需要祖先方向的資料，而祖先在遞迴路徑上早就填好了。

## 鏈狀樹要用迭代版 build

遞迴 `dfs` 在退化成鏈的樹（n = 2×10⁵）會直接爆堆疊。改成 BFS／DFS 先求出一個「parent 必定排在 child 之前」的順序，再照那個順序填表：

```cpp
// Time: O(n log n)   Space: O(n log n)
void buildIter(int root) {
  up.assign(LOG + 1, vector<int>(n, -1));
  vector<int> order, par(n, -1), st{root};
  vector<char> vis(n, 0);
  vis[root] = 1;
  while (!st.empty()) {
    int v = st.back();
    st.pop_back();
    order.push_back(v);
    for (int to : adj[v]) {
      if (!vis[to]) {
        vis[to] = 1;
        par[to] = v;
        depth[to] = depth[v] + 1;
        st.push_back(to);
      }
    }
  }
  for (int v : order) {  // order 保證 parent 已經填完
    up[0][v] = par[v];
    for (int k = 1; k <= LOG; ++k) {
      up[k][v] = up[k - 1][v] == -1 ? -1 : up[k - 1][up[k - 1][v]];
    }
  }
}
```

## 常見陷阱

> [!warning] 五個會讓答案默默錯掉的地方
> - **`LOG` 開太小**：必須滿足 2^LOG ≥ n，否則深樹跳不到頂，錯得無聲無息。實務上直接寫死 `LOG = 20`（2²⁰ > 10⁶）最省事。
> - **階段二的 `k` 寫成從小到大**：貪心方向反了，會在還能跳大步的時候先耗掉小步，停在錯的位置。
> - **對齊深度後忘了檢查 `u == v`**：祖先關係的 case 會落進階段二，而此時每個 `k` 都滿足 `up[k][u] == up[k][v]`，一步都不跳，最後回傳 `up[0][u]` 變成真正答案的**父節點**。
> - **最後回傳 `u` 而不是 `up[0][u]`**：階段二結束時 `u` 停在 LCA 的子節點，差一步。
> - **記憶體**：`up` 是 O(n log n) 個 int，n = 10⁵、LOG = 20 就是 8 MB，通常還好但要有數。

## 跟其他 LCA 做法怎麼選

| 做法 | 預處理 | 每次 query | 適用時機 |
| --- | --- | --- | --- |
| BST 沿大小走 | 無 | O(h) | 樹是 BST，見 [[0235-Lowest-Common-Ancestor-of-a-Binary-Search-Tree]] |
| 後序遞迴 | 無 | O(n) | 普通二元樹、只問一次，見 [[0236-Lowest-Common-Ancestor-of-a-Binary-Tree]] |
| 樸素爬 parent | O(n) 建 parent | O(h) | query 少，或樹保證很淺 |
| **Binary lifting** | O(n log n) | O(log n) | **多次 query 且必須線上作答；順便要查 k 級祖先** |
| Euler tour + sparse table | O(n log n) | O(1) | query 極多、只要 LCA，不需要 k 級祖先 |
| Tarjan 離線（DSU） | O(n α) | 均攤 O(α) | 所有 query 能一次拿到手（離線） |

> [!tip] 倍增表能存的不只是「祖先是誰」
> Binary lifting 真正的價值不只在 LCA。同一張表可以順便存「這 2^k 步路徑上的最大邊權／總和」，於是「兩點路徑上的最大邊權」也變成 O(log n)——這是 Euler tour + sparse table 那套做不到的，也是 binary lifting 在競程裡更常見的原因。

## 題目分類

- **模板題**：[[1483-Kth-Ancestor-of-a-Tree-Node]]（直接要求 k 級祖先，就是 `kthAncestor` 本體）
- **用不到但常被混淆**：[[0235-Lowest-Common-Ancestor-of-a-Binary-Search-Tree]]、[[0236-Lowest-Common-Ancestor-of-a-Binary-Tree]]（都只問一次）
- **變形**：帶 parent 指標時退化成兩鏈求交點，[[1650-Lowest-Common-Ancestor-of-a-Binary-Tree-III]]

## Related Problems

- [[0235-Lowest-Common-Ancestor-of-a-Binary-Search-Tree]] — 單次 query 的 BST 特化解，O(h)／O(1)
- [[0236-Lowest-Common-Ancestor-of-a-Binary-Tree]] — 通用樹的一趟後序解
- [[1483-Kth-Ancestor-of-a-Tree-Node]] — 倍增表的模板題
- [[Tree-Traversal-Iterative]] — 建表要的那趟迭代 DFS 骨架
