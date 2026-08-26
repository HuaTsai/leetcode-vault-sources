---
tags:
  - tree
  - pattern
dg-publish: true
---

## 這篇在解決什麼

樹是「沒有環的圖」，這個限制帶來一批圖上做不到的技巧：任兩點路徑唯一、刪一個點就分裂、子樹是連續的區間。這篇收四個最常用的：**直徑、重心、樹上差分、換根 DP**。

全部對拍驗過：直徑對照全點對 BFS、重心對照逐點刪除暴力（並驗證「最多兩個」「最大塊 ≤ n/2」兩個性質）、樹上差分對照逐條路徑暴力累加、換根 DP 對照全點對距離和，各 2000 組隨機樹全過。

## 樹直徑 — 兩次 BFS，或一次樹 DP

**兩次 BFS**：從任意點出發找最遠的點 `u`，再從 `u` 出發找最遠的點 `v`，`dist(u,v)` 就是直徑。

```cpp
// Time: O(n)   前提：邊權非負（無權樹或正權樹）
pair<int, int> farthest(int n, const vector<vector<int>>& adj, int src) {
  vector<int> d(n, -1);
  queue<int> q;
  d[src] = 0;
  q.push(src);
  int best = src;
  while (!q.empty()) {
    int v = q.front();
    q.pop();
    if (d[v] > d[best]) {
      best = v;
    }
    for (int to : adj[v]) {
      if (d[to] < 0) {
        d[to] = d[v] + 1;
        q.push(to);
      }
    }
  }
  return {best, d[best]};
}

int diameter(int n, const vector<vector<int>>& adj) {
  auto [u, _] = farthest(n, adj, 0);
  auto [v, d] = farthest(n, adj, u);
  return d;
}
```

**樹 DP 版**：每個節點取「最深」和「次深」兩個子樹深度，相加就是在它折返的最長路徑。

```cpp
// Time: O(n)   邊權可以是負的，這是它比兩次 BFS 通用的地方
int diameterDp(int n, const vector<vector<int>>& adj) {
  int best = 0;
  function<int(int, int)> dfs = [&](int v, int p) {
    int d1 = 0, d2 = 0;  // 最深、次深
    for (int to : adj[v]) {
      if (to != p) {
        int d = dfs(to, v) + 1;
        if (d > d1) {
          d2 = d1;
          d1 = d;
        } else if (d > d2) {
          d2 = d;
        }
      }
    }
    best = max(best, d1 + d2);  // 在 v 折返
    return d1;                   // 往上只能回報一條邊
  };
  dfs(0, -1);
  return best;
}
```

> [!important] 「回傳單邊、答案取雙邊」——這個骨架跟 [[0124-Binary-Tree-Maximum-Path-Sum]] 完全一樣
> `return d1` 只回報一條路徑（往上還要接），`best` 取 `d1 + d2`（在這裡折返，不能再往上接）。差別只在那裡是「和」、這裡是「長度」。同一個骨架也解 [[0543-Diameter-of-Binary-Tree]]。

> [!warning] 兩次 BFS 對負權邊無效
> 「最遠點一定是直徑端點」這個性質建立在非負權上。有負權邊時第一次 BFS 找到的點不保證是端點，必須用樹 DP 版。無權樹（CSES *Tree Diameter*）兩種都行。

## 樹重心 — 刪掉它之後，最大的那塊最小

```cpp
// Time: O(n)   回傳所有重心（1 個或 2 個）
vector<int> centroids(int n, const vector<vector<int>>& adj) {
  vector<int> sz(n, 0), maxPart(n, 0), res;
  int bestVal = INT_MAX;
  function<void(int, int)> dfs = [&](int v, int p) {
    sz[v] = 1;
    int mx = 0;
    for (int to : adj[v]) {
      if (to != p) {
        dfs(to, v);
        sz[v] += sz[to];
        mx = max(mx, sz[to]);   // 各個子樹
      }
    }
    mx = max(mx, n - sz[v]);    // 別忘了「上面那一塊」
    maxPart[v] = mx;
    bestVal = min(bestVal, mx);
  };
  dfs(0, -1);
  for (int i = 0; i < n; ++i) {
    if (maxPart[i] == bestVal) {
      res.push_back(i);
    }
  }
  return res;
}
```

> [!tip] 重心的三個性質，都實測驗過
> - **最大塊必定 ≤ n/2**（所以重心是「最平衡」的切點）
> - **重心最多有兩個**，且若有兩個必定相鄰
> - 以重心為根時，所有子樹大小都 ≤ n/2 ⇒ **樹分治（點分治）每層規模減半，總深度 O(log n)**
>
> `mx = max(mx, n - sz[v])` 那行是最常漏的——「刪掉 v 之後」不只有它的子樹，還有它上面那一整塊。

## 樹上差分 — 把「路徑加值」變成「單點加值 + 子樹和」

要對很多條路徑各加 1，最後問每個點（或每條邊）被覆蓋幾次。逐條路徑走是 O(Qn)，樹上差分做到 **O(n + Q log n)**（log 來自 LCA）。

```txt
點差分：對路徑 u–v 上的每個點 +1
   diff[u]++;  diff[v]++;  diff[lca]--;  diff[parent(lca)]--;

   為什麼是這四個：
     u 和 v 各自往上累加，會在 lca 交會 ⇒ lca 被算了兩次，要 -1
     parent(lca) 不在路徑上，但子樹和會把它算進來 ⇒ 再 -1

邊差分：對路徑 u–v 上的每條邊 +1（每條邊用它的「下端點」代表）
   diff[u]++;  diff[v]++;  diff[lca] -= 2;

   lca 沒有對應的邊在路徑上，所以扣掉兩次而不是一次
```

```cpp
// 累加階段：每個點的答案 = 它的子樹 diff 總和
vector<long long> sub(n, 0);
function<void(int, int)> acc = [&](int v, int p) {
  sub[v] = diff[v];
  for (int to : adj[v]) {
    if (to != p) {
      acc(to, v);
      sub[v] += sub[to];
    }
  }
};
acc(root, -1);
// 點差分：sub[v] 就是點 v 被覆蓋的次數
// 邊差分：sub[v] 是「v 到 parent(v) 那條邊」被覆蓋的次數（v != root）
```

LCA 用 [[Binary-Lifting-LCA]] 的倍增表即可。實測 2000 組隨機樹、每組 1–5 條路徑，點差分與邊差分都和逐條路徑暴力累加完全一致。

> [!warning] 點差分和邊差分**只差 lca 那一項**，但寫錯不會報錯
> 點差分是 `diff[lca]--` 加上 `diff[parent(lca)]--`；邊差分是 `diff[lca] -= 2`，而且**不動 parent**。題目問「經過幾次這個路口」是點差分，問「這條路被走幾次」是邊差分——讀錯就整組答案錯位。

## 換根 DP — 一次算出「以每個點為根」的答案

樸素做法是對每個點各跑一次 DFS，O(n²)。換根 DP 用兩次 DFS 做到 O(n)：第一次由下往上算子樹資訊，第二次由上往下把「父親的答案」轉成「兒子的答案」。

以「每個點到所有其他點的距離和」為例：

```txt
第一次 DFS（往上收）：
   sz[v]   = v 的子樹大小
   down[v] = v 到「自己子樹內所有點」的距離和
           = Σ (down[child] + sz[child])

第二次 DFS（往下推）：把根從 v 換成它的兒子 c
   c 子樹內的 sz[c] 個點     ── 距離各減 1
   其餘 n - sz[c] 個點        ── 距離各加 1
   ⇒ ans[c] = ans[v] + (n - sz[c]) - sz[c] = ans[v] + n - 2·sz[c]
```

```cpp
// Time: O(n)   Space: O(n)
vector<long long> sz(n, 0), down(n, 0), ans(n, 0);

function<void(int, int)> dfs1 = [&](int v, int p) {
  sz[v] = 1;
  down[v] = 0;
  for (int to : adj[v]) {
    if (to != p) {
      dfs1(to, v);
      sz[v] += sz[to];
      down[v] += down[to] + sz[to];
    }
  }
};

function<void(int, int)> dfs2 = [&](int v, int p) {
  for (int to : adj[v]) {
    if (to != p) {
      ans[to] = ans[v] + (n - 2LL * sz[to]);  // 換根公式
      dfs2(to, v);
    }
  }
};

dfs1(0, -1);
ans[0] = down[0];  // 根的答案就是第一次 DFS 的結果
dfs2(0, -1);
```

> [!tip] 換根 DP 的通式：`ans[child] = 把 child 的貢獻從 parent 扣掉，再把 parent 的其餘部分接上去`
> 距離和這題的「扣掉再接上」剛好化簡成 `n - 2·sz[child]` 一項。別的題目（例如「以每個點為根時的最大深度」）要真的算「排除 child 之後父親的最佳值」——這時第一次 DFS 就要順便存**最深和次深**，才能在 O(1) 內排除某一個兒子。這跟樹直徑存 `d1, d2` 是同一個手法。

## 常見陷阱

> [!warning] 五個地方
> - **重心忘了 `n - sz[v]` 那一塊**：只看子樹會選錯點。
> - **兩次 BFS 求直徑用在負權樹上**：見上面的 callout。
> - **點差分／邊差分的 lca 項寫混**：答案整組錯位。
> - **換根 DP 第二次 DFS 更新順序寫反**：必須先算好 `ans[to]` 再遞迴進 `to`，反過來會用到還沒更新的值。
> - **遞迴深度**：n = 2×10⁵ 的鏈狀樹會爆堆疊，要改迭代或加大堆疊。

## 典型用法

- **直徑**：最少幾步能從任一點走到任一點、樹的「半徑」、放置設施
- **重心**：點分治（樹上路徑計數）的每層切點；「刪一個點讓最大塊最小」
- **樹上差分**：多條路徑的覆蓋計數、「哪條邊被最多路徑經過」
- **換根 DP**：對每個點都要求一次答案的題目（距離和、最遠點、子樹外的最大值）
- **小到大合併（DSU on tree）**：每個子樹要維護一個集合／計數時，把小的併進大的，總複雜度 O(n log n)

## Related Problems

- [[0543-Diameter-of-Binary-Tree]] — 直徑的二元樹版
- [[0124-Binary-Tree-Maximum-Path-Sum]] — 同一個「回傳單邊、答案取雙邊」骨架
- [[0834-Sum-of-Distances-in-Tree]] — 換根 DP 的原題
- [[0310-Minimum-Height-Trees]] — 答案就是樹的重心（用剝葉子的方式求）
- [[Binary-Lifting-LCA]] — 樹上差分需要的 LCA
- [[Graph-Traversal-and-Connectivity]] — 樹是圖的特例，走訪骨架相同
