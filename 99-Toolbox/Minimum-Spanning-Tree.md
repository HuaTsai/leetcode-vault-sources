---
tags:
  - graph
  - pattern
dg-publish: true
---

## 這篇在解決什麼

「用最小的總成本把所有點連起來」——連接城市、鋪電纜、[[1584-Min-Cost-to-Connect-All-Points]]。答案一定是一棵**生成樹**（n 個點、n-1 條邊、連通、無環），要找的是權重和最小的那棵。

兩個標準解法都短，選哪個只看圖是稀疏還是稠密。這篇兩種都收，並對照 2ᵐ 暴力枚舉所有邊子集驗過。

## 核心：切割性質

> 把所有點任意分成兩堆，**跨越這兩堆的邊裡權重最小的那條，一定在某棵最小生成樹上**。

Kruskal 和 Prim 都只是這條性質的不同用法：

```txt
Kruskal：邊由小到大掃，只要這條邊的兩端還沒連通就收
         （「還沒連通」＝ 它跨越了某個切割，而它是目前最小的跨越邊）

Prim：   從一個點開始長，每次挑「已選集合」到「未選集合」的最小邊
         （切割固定為「已選 vs 未選」，每次取最小跨越邊）
```

## Kruskal — 排序 + [[Disjoint-Set-Union]]

`DSU` 的完整實作在 [[Disjoint-Set-Union]]，這裡只用到它的 `unite`（回傳 `false` 代表兩端本來就連通）。

```cpp
// Time: O(m log m)（瓶頸是排序）   Space: O(n + m)
struct WEdge {
  int u, v;
  long long w;
};

// 回傳 {總權重, 用了幾條邊}；used < n-1 代表圖不連通、沒有生成樹
pair<long long, int> kruskal(int n, vector<WEdge> edges) {
  sort(edges.begin(), edges.end(),
       [](const WEdge& a, const WEdge& b) { return a.w < b.w; });

  DSU d(n);
  long long total = 0;
  int used = 0;
  for (const auto& e : edges) {
    if (d.unite(e.u, e.v)) {  // 回傳 false 代表兩端已連通，這條邊會造成環
      total += e.w;
      ++used;
    }
  }
  return {total, used};
}
```

> [!important] 一定要檢查 `used == n - 1`
> 圖不連通時 Kruskal 不會報錯，它會安靜地回傳一片**森林**的權重和。很多題目保證連通所以看不出來，但 CSES 的 *Road Reparation* 這類題就是在考這個——答案要輸出 `IMPOSSIBLE`。`used` 這個回傳值就是為此存在。

## Prim — 從一個點長出去

```cpp
// Time: O(m log n)   Space: O(n + m)
// 回傳 {總權重, 用了幾條邊}
pair<long long, int> prim(int n, const vector<vector<pair<int, long long>>>& adj) {
  vector<char> inTree(n, 0);
  priority_queue<pair<long long, int>, vector<pair<long long, int>>, greater<>> pq;
  pq.push({0, 0});  // 從節點 0 開始，第一條「邊」權重算 0

  long long total = 0;
  int cnt = 0;
  while (!pq.empty()) {
    auto [w, v] = pq.top();
    pq.pop();
    if (inTree[v]) {
      continue;  // 懶刪除，跟 Dijkstra 同一招
    }
    inTree[v] = 1;
    total += w;
    ++cnt;
    for (auto [to, ww] : adj[v]) {
      if (!inTree[to]) {
        pq.push({ww, to});
      }
    }
  }
  return {total, cnt - 1};  // cnt 是進樹的點數，邊數少一
}
```

> [!tip] Prim 和 Dijkstra 只差一個字
> 骨架完全一樣（priority_queue + 懶刪除），差別只在放進 pq 的東西：
>
> ```txt
> Dijkstra： pq.push({dist[v] + w, to})   ← 從起點算起的累積距離
> Prim：     pq.push({w, to})             ← 只有這一條邊的權重
> ```
>
> 一個要「離起點多遠」，一個要「接進來多貴」。記住這個對照，兩個就不會寫混。

## 怎麼選

| | Kruskal | Prim |
| --- | --- | --- |
| 複雜度 | O(m log m) | O(m log n) |
| 圖的表示 | 邊集 | 鄰接表 |
| 稀疏圖（m ≈ n） | **較好**，排序快 | 也可以 |
| 稠密圖（m ≈ n²） | 排序 n² 條邊較慢 | **較好**；配 O(n²) 樸素版更好 |
| 已經寫了 DSU | 直接用 | — |
| 要邊排序做別的事 | 順便 | — |

實務上**預設用 Kruskal**：短、直觀、DSU 通常已經有了，而且「邊按權排序」這個中間結果常常題目還要拿去做別的事。稠密圖（n ≤ 1000 但邊接近滿）才換 Prim。

## 常見陷阱

> [!warning] 五個會出錯的地方
> - **沒檢查連通性**：不連通時回傳森林權重，見上面的 callout。
> - **權重用 `int` 累加**：n = 2×10⁵ 條邊、每條 10⁹，總和 10¹⁴。`long long`。
> - **無向圖只加了單向邊**：Prim 走鄰接表，漏掉反向邊會少長出一半的樹。Kruskal 用邊集所以沒這問題——這也是它比較不容易寫錯的原因之一。
> - **Prim 忘記懶刪除的 `continue`**：同一個點被重複計入 `total`。
> - **以為 MST 唯一**：權重有相同值時 MST 可能有多棵，**但總權重唯一**。題目若要求「輸出方案」得小心，要求「輸出總和」就沒差。

## 變形

- **最大生成樹**：權重取負，或排序改成由大到小。Kruskal 一個字就改完。
- **最小瓶頸路**（讓路徑上最大邊權最小）：**MST 上的路徑就是答案**。Kruskal 加邊到兩點連通為止，最後加的那條邊的權重就是瓶頸值。也可以用 Dijkstra 把 `dist[v] + w` 換成 `max(dist[v], w)`，見 [[Shortest-Path-Algorithms]]。
- **第二小生成樹**：先求 MST，再枚舉每條非樹邊，換掉它在樹上路徑的最大邊——那個路徑最大邊用 [[Binary-Lifting-LCA]] 的倍增表 O(log n) 查。
- **有 k 個點必須是不同分量**：跑 Kruskal 但只合併到剩 k 個連通塊就停（分群問題）。
- **虛擬節點**：「可以在某點自建水井（成本 c）或連管線」這種題，開一個虛擬點連到所有可自建的點、權重 c，然後跑普通 MST。

## Related Problems

- [[1584-Min-Cost-to-Connect-All-Points]] — 完全圖上的 MST，稠密，適合 Prim
- [[1135-Connecting-Cities-With-Minimum-Cost]] — Kruskal 原型，要判不連通
- [[1489-Find-Critical-and-Pseudo-Critical-Edges-in-MST]] — 反覆跑 Kruskal 判關鍵邊
- [[1631-Path-With-Minimum-Effort]] — 最小瓶頸路，MST 或 Dijkstra 變形都行
- [[Disjoint-Set-Union]] — Kruskal 的地基
- [[Shortest-Path-Algorithms]] — Prim 和 Dijkstra 的對照
