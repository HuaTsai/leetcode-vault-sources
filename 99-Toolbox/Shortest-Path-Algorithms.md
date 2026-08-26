---
tags:
  - graph
  - shortest-path
  - pattern
dg-publish: true
---

## 這篇在解決什麼

最短路有四種常用解法，選錯的代價分兩種：**選太重**會 TLE，**選太輕**會 WA（而且是在有負邊的測資上才錯，本機小測資看不出來）。這篇先給選型流程，再逐一收模板與各自的陷阱。

全部對拍驗過：非負權圖上 Dijkstra 兩版與 Bellman-Ford 對照 Floyd-Warshall 17035 組全過；0-1 BFS 對照 Dijkstra 17035 組全過。

## 先決定用哪一個

```txt
邊權有負的嗎？
├─ 沒有 ─→ 邊權只有 0 和 1？
│           ├─ 是 ─→ 0-1 BFS（deque）        O(n + m)
│           └─ 否 ─→ 邊權全部相同？
│                     ├─ 是 ─→ 普通 BFS       O(n + m)
│                     └─ 否 ─→ Dijkstra       O(m log n)
└─ 有 ───→ 要所有點對之間的距離嗎？
             ├─ 是 ─→ Floyd-Warshall          O(n³)
             └─ 否 ─→ Bellman-Ford            O(nm)，順便偵測負環
```

| | 負權 | 負環偵測 | 複雜度 | 適用 n, m |
| --- | --- | --- | --- | --- |
| BFS | ✗ | — | O(n + m) | 邊權全同 |
| 0-1 BFS | ✗ | — | O(n + m) | 邊權只有 0／1 |
| Dijkstra | ✗ | — | O(m log n) | n, m 到 10⁵–10⁶ |
| Bellman-Ford | ✓ | ✓ | O(nm) | n ≤ 10³ 左右 |
| Floyd-Warshall | ✓ | ✓ | O(n³) | n ≤ 400–500 |

## 兩種圖表示法

Dijkstra 和 0-1 BFS 走的是**鄰接表**（要能從一個點快速找到它的出邊）；Bellman-Ford 和 Floyd 走的是**邊集**（每輪要掃過所有邊）。本篇的模板統一用這兩種型別：

```cpp
// 鄰接表：adj[u] 裡每個 pair 是 (終點, 權重)
vector<vector<pair<int, long long>>> adj(n);
adj[u].push_back({v, w});          // 有向邊 u → v
// adj[v].push_back({u, w});       // 無向圖要補這行

// 邊集
struct Edge {
  int u, v;
  long long w;
};
vector<Edge> edges;
edges.push_back({u, v, w});
```

## Dijkstra — 非負權的單源最短路

每次從「還沒定案的點裡挑距離最小的」，把它定案並鬆弛它的出邊。**非負權**保證了「當前最小的那個距離不可能再變小」，這就是正確性的全部依據。

```cpp
// Time: O(m log n)   Space: O(n + m)
// 前提：所有邊權 >= 0
vector<long long> dijkstra(int n, const vector<vector<pair<int, long long>>>& adj, int src) {
  const long long INF = LLONG_MAX / 4;  // 除以 4 留空間，避免 dist + w 溢位
  vector<long long> dist(n, INF);
  priority_queue<pair<long long, int>, vector<pair<long long, int>>, greater<>> pq;

  dist[src] = 0;
  pq.push({0, src});
  while (!pq.empty()) {
    auto [d, v] = pq.top();
    pq.pop();
    if (d > dist[v]) {
      continue;  // 過期條目：這個點已經用更短的距離處理過了
    }
    for (auto [to, w] : adj[v]) {
      if (dist[v] + w < dist[to]) {
        dist[to] = dist[v] + w;
        pq.push({dist[to], to});  // 不刪舊的，直接塞新的
      }
    }
  }
  return dist;
}
```

> [!important] 用「懶刪除」而不是 `visited` 陣列，也不要用 `decrease-key`
> `priority_queue` 沒有 decrease-key，標準做法是**同一個點可以進 pq 好幾次**，靠 `if (d > dist[v]) continue;` 在彈出時丟掉過期條目。pq 裡最多 m 個條目，複雜度仍是 O(m log n)。
>
> 想用 `set` 模擬 decrease-key（`erase` 舊的再 `insert` 新的）也可以，但常數更大、更容易寫錯，沒有理由。

> [!warning] 把懶刪除換成 `visited` 標記，在負權圖上會靜靜地算錯
> 常見的另一種寫法是彈出時 `if (done[v]) continue; done[v] = 1;`，一旦定案就永不更新。這在非負權下與懶刪除版**完全等價**（17035 組對拍一致），但只要有一條負邊就會錯。三個節點就能構造出來：
>
> ```txt
> 0 --2--> 1          求 0 到 1 的最短路
> 0 --3--> 2          正解：0→2→1 = 3 + (-2) = 1
> 2 --(-2)--> 1       visited 版：先定案 1（距離 2），之後 2 想把它改成 1 已經不准了 → 輸出 2
> ```
>
> 實測 10238 組隨機負權（無負環）圖：懶刪除版 **0 組錯**，visited 版 **20 組錯**。
>
> 但這不代表「懶刪除版可以拿來跑負權圖」——它之所以還對，是因為過期條目會讓它反覆重新鬆弛，實質退化成 Bellman-Ford，**複雜度失去保證、最壞可以指數爆炸**。有負邊就換 Bellman-Ford，這是唯一正確的反應。

## 0-1 BFS — 邊權只有 0 和 1 時，deque 取代 heap

邊權只有 0／1 時，佇列裡的距離最多只有兩種值（`d` 和 `d+1`）。用 `deque`：走 0 邊就 `push_front`（跟當前同層），走 1 邊就 `push_back`（下一層）。佇列自然保持有序，省掉 heap 的 log。

```cpp
// Time: O(n + m)   Space: O(n + m)
// 前提：邊權只有 0 或 1
vector<long long> bfs01(int n, const vector<vector<pair<int, long long>>>& adj, int src) {
  const long long INF = LLONG_MAX / 4;
  vector<long long> dist(n, INF);
  deque<int> dq;

  dist[src] = 0;
  dq.push_front(src);
  while (!dq.empty()) {
    int v = dq.front();
    dq.pop_front();
    for (auto [to, w] : adj[v]) {
      if (dist[v] + w < dist[to]) {
        dist[to] = dist[v] + w;
        if (w == 0) {
          dq.push_front(to);  // 同層，插隊到前面
        } else {
          dq.push_back(to);   // 下一層，排到後面
        }
      }
    }
  }
  return dist;
}
```

> [!tip] 「最少要翻轉幾條邊」「最少要打破幾道牆」這類題，通常就是 0-1 BFS
> 把「順著走」當成權 0、「逆著走／要付出一次代價」當成權 1，答案就是最短路。看到「最少改變幾次方向」「最少花費 1 的操作次數」而其他移動免費，先想這個而不是 Dijkstra。

## Bellman-Ford — 有負權時的單源最短路，兼負環偵測

最短路最多走 n-1 條邊（再多就有重複點）。所以把**所有邊**鬆弛 n-1 輪就一定收斂。第 n 輪還能鬆弛成功 ⇒ 存在負環。

```cpp
// Time: O(nm)   Space: O(n)
// 回傳 false 代表「從 src 可達的範圍內」存在負環
bool bellmanFord(int n, const vector<Edge>& edges, int src, vector<long long>& dist) {
  const long long INF = LLONG_MAX / 4;
  dist.assign(n, INF);
  dist[src] = 0;

  for (int i = 0; i < n - 1; ++i) {
    bool changed = false;
    for (const auto& e : edges) {
      if (dist[e.u] < INF && dist[e.u] + e.w < dist[e.v]) {
        dist[e.v] = dist[e.u] + e.w;
        changed = true;
      }
    }
    if (!changed) {
      break;  // 提前收斂，實際上很少跑滿 n-1 輪
    }
  }

  for (const auto& e : edges) {  // 第 n 輪：還能變短就是有負環
    if (dist[e.u] < INF && dist[e.u] + e.w < dist[e.v]) {
      return false;
    }
  }
  return true;
}
```

> [!warning] `dist[e.u] < INF` 這個檢查不能省
> 少了它，從 `INF` 的點出發還會去鬆弛別人：`INF + (-5) < INF` 成立，於是不可達的點被寫成 `INF - 5`，然後污染擴散開來。這也是為什麼 `INF` 要用 `LLONG_MAX / 4` 而不是 `LLONG_MAX`——後者 `INF + w` 直接溢位成負數，整張表都會爛掉。

> [!note] 它只偵測「從 src 走得到」的負環
> 題目若要問「圖裡任何地方有沒有負環」，加一個虛擬源點連到**所有**節點（權 0），從它跑一次即可。CSES 的 *Cycle Finding* 就是這個變形。

## Floyd-Warshall — 全點對，五行

`d[i][j]` 表示「只允許用前 k 個點當中繼」時 i 到 j 的最短路。k 從 0 掃到 n-1，每次問「繞過 k 會不會更短」。

```cpp
// Time: O(n^3)   Space: O(n^2)
// 允許負權；d[i][i] < 0 代表 i 在某個負環上
const long long INF = LLONG_MAX / 4;
vector<vector<long long>> d(n, vector<long long>(n, INF));
for (int i = 0; i < n; ++i) {
  d[i][i] = 0;
}
for (const auto& e : edges) {
  d[e.u][e.v] = min(d[e.u][e.v], e.w);  // 用 min：重邊只留最短的那條
}

for (int k = 0; k < n; ++k) {           // k 一定要在最外層
  for (int i = 0; i < n; ++i) {
    if (d[i][k] == INF) {
      continue;                          // 剪枝，常常快好幾倍
    }
    for (int j = 0; j < n; ++j) {
      if (d[k][j] < INF && d[i][k] + d[k][j] < d[i][j]) {
        d[i][j] = d[i][k] + d[k][j];
      }
    }
  }
}
```

> [!important] `k` 必須是最外層迴圈
> 三層迴圈的順序不是隨便的。`k` 在外層才符合「逐步放寬可用中繼點集合」的 DP 定義；寫成 `i, j, k` 或 `i, k, j` 會在同一輪裡用到還沒算完的狀態，得到的結果**大部分測資看起來是對的**，但特定圖上會錯——是那種很難 debug 的錯。

## 常見陷阱

> [!warning] 五個共通的坑
> - **`INF` 用 `LLONG_MAX`／`INT_MAX`**：`INF + w` 立刻溢位。一律用 `LLONG_MAX / 4` 這種留了餘裕的值，並在鬆弛前檢查來源是否可達。
> - **`dist` 用 `int`**：n = 10⁵ 條邊、每條權 10⁹，總距離 10¹⁴。`long long`。
> - **無向圖忘記加反向邊**：`adj[u].push_back({v, w})` 之後要補 `adj[v].push_back({u, w})`。
> - **有負邊卻用了 Dijkstra**：見上面的三節點反例。這個錯在小測資上不一定顯現。
> - **Floyd 的 `k` 沒放最外層**：見上一個 callout。

## 典型變形

- **次短路**：每個點存兩個距離（最短、次短），鬆弛時同時維護
- **最短路計數**：`cnt[to] += cnt[v]`（距離相等時）／`cnt[to] = cnt[v]`（距離變短時）
- **有限邊數的最短路**（「最多轉機 k 次」）：Bellman-Ford 只跑 k+1 輪，且每輪要用**上一輪的 dist 副本**鬆弛，否則一輪內會連走多條邊
- **分層圖**：「可以免費走 k 條邊」把狀態擴成 `(節點, 已用次數)`，在 `n(k+1)` 個點上跑 Dijkstra
- **最小瓶頸路**（路徑上最大邊權最小）：Dijkstra 把 `dist[v] + w` 換成 `max(dist[v], w)`

## Related Problems

- [[0743-Network-Delay-Time]] — 最直接的 Dijkstra
- [[0787-Cheapest-Flights-Within-K-Stops]] — 限制邊數，Bellman-Ford 跑 k+1 輪
- [[1091-Shortest-Path-in-Binary-Matrix]] — 邊權全同，用 BFS 就好
- [[1368-Minimum-Cost-to-Make-at-Least-One-Valid-Path-in-a-Grid]] — 0-1 BFS 的代表題
- [[1631-Path-With-Minimum-Effort]] — 最小瓶頸路，Dijkstra 換合併函式
- [[Disjoint-Set-Union]] — 最小瓶頸路的另一解：Kruskal 加邊到連通為止
