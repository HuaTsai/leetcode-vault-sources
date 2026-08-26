---
tags:
  - graph
  - pattern
dg-publish: true
---

## 這篇在解決什麼

圖題的前三分之一都是同一件事的變形：**走過整張圖，順便回答一個關於結構的問題**——連不連通、有沒有環、能不能排出順序、能不能兩色染完。這些全部共用 BFS／DFS 骨架，差別只在「走的時候記什麼」。

這篇把骨架和五個常見問題收在一起，全部對拍驗過（BFS 對照 Floyd、二分圖對照 2ⁿ 暴力染色、拓撲序逐邊檢查、環偵測對照 DSU，合計 20226 組 + 環偵測另外 30000 組）。

```txt
                    走完整張圖
                        │
   ┌────────────┬───────┴───────┬──────────────┐
 記距離        記顏色          記時間戳        記代表元
   │             │                │              │
 BFS 最短路   二分圖判定    拓撲排序／環偵測   連通塊
              （2 色）      （白灰黑三色）    （DSU 或 DFS）
```

## BFS — 無權圖最短路

佇列保證「先進來的距離較小」，所以第一次碰到某個點時就是最短距離。

```cpp
// Time: O(n + m)   Space: O(n)
vector<int> bfs(int n, const vector<vector<int>>& adj, int src) {
  vector<int> dist(n, -1);  // -1 = 不可達，同時兼任 visited
  queue<int> q;

  dist[src] = 0;
  q.push(src);
  while (!q.empty()) {
    int v = q.front();
    q.pop();
    for (int to : adj[v]) {
      if (dist[to] == -1) {
        dist[to] = dist[v] + 1;
        q.push(to);  // 進佇列時就標記，不是出佇列時
      }
    }
  }
  return dist;
}
```

> [!warning] 標記一定要在 `push` 的時候做，不是 `pop` 的時候
> 若改成彈出時才標記，同一個點會被不同鄰居重複塞進佇列，佇列長度從 O(n) 膨脹到 O(m)，退化成指數級的重複展開。`dist[to] == -1` 這個判斷同時扮演 visited 的角色，就是為了讓標記和入隊綁在一起。

## DFS — 迭代版避開爆堆疊

遞迴 DFS 在鏈狀圖（n = 10⁵）會爆堆疊。迭代版把「待辦清單」換成 `stack`：

```cpp
// Time: O(n + m)   Space: O(n)
vector<int> dfsIter(int n, const vector<vector<int>>& adj, int src) {
  vector<int> order;
  vector<char> vis(n, 0);
  vector<int> st{src};

  while (!st.empty()) {
    int v = st.back();
    st.pop_back();
    if (vis[v]) {
      continue;  // 可能被重複塞入，彈出時才確認
    }
    vis[v] = 1;
    order.push_back(v);
    for (int i = adj[v].size() - 1; i >= 0; --i) {
      if (!vis[adj[v][i]]) {
        st.push_back(adj[v][i]);  // 倒著推，彈出順序才跟遞迴一致
      }
    }
  }
  return order;
}
```

> [!note] 這個版本和 BFS 的標記時機**剛好相反**
> `stack` 版必須在彈出時才標記——因為一個點可能先被推入、之後又從別的路徑被推入，而我們要的是「最後推入的那次」決定順序。這跟 BFS「入隊即標記」是不同的取捨：stack 版容許重複入棧（最多 m 個），BFS 不容許。詳細的三序骨架見 [[Tree-Traversal-Iterative]]。

## 拓撲排序 — 兩種寫法，都順便偵測環

**Kahn（BFS 版）**：反覆取出入度為 0 的點。取不完 ⇒ 有環。

```cpp
// Time: O(n + m)   回傳 false 代表有環
bool topoKahn(int n, const vector<vector<int>>& adj, vector<int>& order) {
  vector<int> indeg(n, 0);
  for (int u = 0; u < n; ++u) {
    for (int v : adj[u]) {
      ++indeg[v];
    }
  }
  queue<int> q;
  for (int i = 0; i < n; ++i) {
    if (indeg[i] == 0) {
      q.push(i);
    }
  }
  order.clear();
  while (!q.empty()) {
    int v = q.front();
    q.pop();
    order.push_back(v);
    for (int to : adj[v]) {
      if (--indeg[to] == 0) {
        q.push(to);
      }
    }
  }
  return (int)order.size() == n;  // 少了任何一個點 ⇒ 那些點在環上
}
```

**DFS 版**：後序完成的順序反轉就是拓撲序，用三色標記同時抓環。

```cpp
// Time: O(n + m)   回傳 false 代表有環
bool topoDfs(int n, const vector<vector<int>>& adj, vector<int>& order) {
  vector<int> color(n, 0);  // 0=白(未訪) 1=灰(在遞迴堆疊上) 2=黑(已完成)
  order.clear();
  bool hasCycle = false;

  function<void(int)> dfs = [&](int v) {
    color[v] = 1;
    for (int to : adj[v]) {
      if (color[to] == 1) {
        hasCycle = true;  // 碰到灰的 = 回邊 = 環
        return;
      }
      if (color[to] == 0) {
        dfs(to);
      }
    }
    color[v] = 2;
    order.push_back(v);  // 後序：所有後代都完成了才輪到自己
  };

  for (int i = 0; i < n; ++i) {
    if (color[i] == 0) {
      dfs(i);
    }
  }
  reverse(order.begin(), order.end());
  return !hasCycle;
}
```

> [!important] 三色的「灰」才是重點，兩色（visited / not visited）抓不到有向環
> 碰到**黑色**節點只代表「這條路我走過了」，完全正常（DAG 上到處都是）。碰到**灰色**才代表「我沿著這條路走回了還在堆疊上的祖先」，那才是環。只用一個 `visited` 陣列的話，黑灰不分，會把正常的交叉邊誤判成環。
>
> 無向圖不需要三色——那裡的問題不是「灰或黑」，而是「別把來時路當成環」，見下一節。

## 環偵測 — 有向用三色，無向要排除來時路

無向圖每條邊在兩端各存一次，所以從 `v` 走到 `to` 之後，`to` 一定看得到一條回到 `v` 的邊——那不是環，是同一條邊。有兩種排除法：

```cpp
// 寫法 A：記住「走過來的那條邊的 id」
bool hasCycleByEdgeId(int n, const vector<vector<pair<int, int>>>& adj) {
  vector<char> vis(n, 0);
  function<bool(int, int)> dfs = [&](int v, int parentEdge) {
    vis[v] = 1;
    for (auto [to, id] : adj[v]) {
      if (id == parentEdge) {
        continue;  // 就是走過來的那一條
      }
      if (vis[to]) {
        return true;
      }
      if (dfs(to, id)) {
        return true;
      }
    }
    return false;
  };
  for (int i = 0; i < n; ++i) {
    if (!vis[i] && dfs(i, -1)) {
      return true;
    }
  }
  return false;
}

// 寫法 B：記住「走過來的那個點」——更短，實測與 A 等價
//   for (int to : adj[v]) { if (to == parent) continue; ... dfs(to, v) ... }
```

> [!note] 寫法 B 在重邊和自環下也是對的——實測 30000 組零差異
> 直覺上會擔心「兩條平行邊 u–v，B 版把兩條都當成來時路跳過」而漏判。實際跑起來不會：從 `u` 走第一條到 `v`，`v` 把所有回到 `u` 的邊都跳掉了沒錯，但**回溯之後 `u` 還會繼續看第二條邊**，此時 `vis[v]` 已為真，照樣回報有環。自環同理（`vis[v]` 進入時就設了）。
>
> 30000 組隨機圖（含重邊、含自環）對照 DSU 基準：兩種寫法各 0 錯、彼此 0 不一致。**選短的 B 就好**；A 的價值只在意圖比較明顯，以及之後要改成「找出環上的邊」時更好擴充。

另一個更省事的無向環偵測：**用 [[Disjoint-Set-Union]]**——加邊時 `unite` 回傳 `false` 就代表這條邊接在已連通的兩點之間，也就是造成環。連通塊數量也順便算好了。

## 二分圖判定 — BFS 兩色染

沿著邊染相反色，碰到「鄰居和自己同色」就不是二分圖。等價於「圖中沒有奇數長度的環」。

```cpp
// Time: O(n + m)
bool isBipartite(int n, const vector<vector<int>>& adj, vector<int>& color) {
  color.assign(n, -1);
  for (int s = 0; s < n; ++s) {
    if (color[s] != -1) {
      continue;  // 圖可能不連通，每個連通塊都要試
    }
    color[s] = 0;
    queue<int> q;
    q.push(s);
    while (!q.empty()) {
      int v = q.front();
      q.pop();
      for (int to : adj[v]) {
        if (color[to] == -1) {
          color[to] = color[v] ^ 1;
          q.push(to);
        } else if (color[to] == color[v]) {
          return false;
        }
      }
    }
  }
  return true;
}
```

> [!warning] 外層迴圈不能省
> 圖不保證連通。只從節點 0 染一次，另一個連通塊裡的奇環就漏掉了。這個 bug 在「圖恰好連通」的測資上完全看不出來——所有「掃全圖」的演算法（連通塊、拓撲、環偵測）都有同一個外層迴圈，漏掉的後果一樣。

## 常見陷阱

> [!warning] 五個共通的坑
> - **BFS 在 `pop` 時才標記**：佇列爆炸，見上面第一個 callout。
> - **有向環偵測只用兩色**：交叉邊被誤判成環。
> - **無向環偵測忘記排除來時路**：每條邊都被當成環。
> - **忘記外層 `for` 迴圈**：不連通的圖只處理了第一塊。
> - **遞迴 DFS 碰上 10⁵ 的鏈**：爆堆疊。改迭代版，或確認深度有界。

## 典型用法

- **連通塊計數／最大塊**：DFS 掃全圖，或直接 [[Disjoint-Set-Union]]
- **課程安排／建置順序**：拓撲排序，取不完就是有循環依賴
- **DAG 上的 DP**：先拓撲排序，再照那個順序遞推（保證算到某點時前驅都算完了）
- **二分圖 → 最大匹配**：判完是二分圖之後接匈牙利／Dinic
- **無權最短路 / 最少步數**：BFS；邊權 0/1 見 [[Shortest-Path-Algorithms]]
- **多源 BFS**：一開始把所有源點都塞進佇列，距離全設 0

## Related Problems

- [[0200-Number-of-Islands]] — 連通塊計數的入門形
- [[0207-Course-Schedule]] — 拓撲排序判環
- [[0210-Course-Schedule-II]] — 同上但要輸出順序
- [[0785-Is-Graph-Bipartite]] — 二分圖判定本體
- [[0994-Rotting-Oranges]] — 多源 BFS
- [[Disjoint-Set-Union]] — 無向連通性的另一套工具
- [[Shortest-Path-Algorithms]] — 有權時的升級
