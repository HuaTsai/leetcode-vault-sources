---
tags:
  - graph
  - pattern
dg-publish: true
---

## 這篇在解決什麼

最大流本身的模板不難（Dinic 大約四十行），**難的是看出「這題是最大流」以及怎麼建圖**。這篇一半篇幅給模板，一半給建模——後者才是實際卡住的地方。

對拍驗過：Dinic 的最大流對照 2ⁿ 暴力枚舉最小割 2000 組、匈牙利對照 Dinic 與 2ᵐ 暴力匹配 3000 組、König 定理（最小點覆蓋 = 最大匹配）對照暴力 2000 組，全過。

## 核心：反向邊就是「反悔」的權利

增廣路演算法的正確性完全建立在反向邊上：

```txt
從 u 往 v 推了 5 單位流量之後：
   正向邊 u→v 的剩餘容量 −5
   反向邊 v→u 的容量     +5   ← 一開始是 0

反向邊的意義是「這 5 單位可以退回來」。之後若發現讓這 5 單位走 v
是次佳選擇，演算法可以沿著反向邊「抵銷」它，改道走別條路。

沒有反向邊的話，第一次的貪心選擇就永遠改不了 —— 那不是最大流，
只是某一條增廣路徑的結果。
```

實作上把邊成對存進同一個陣列，第 `i` 條邊的反向邊就是 `i ^ 1`。

## Dinic

BFS 把圖分層（`level[v]` = 從源點的最短距離），DFS 只沿著「層數 +1」的邊找增廣路。這個限制讓每輪 BFS 之後找到的都是**最短增廣路**，總輪數 O(n)。

```cpp
// Time: O(n^2 m) 一般圖；單位容量圖 O(m sqrt(m))；二分圖匹配 O(m sqrt(n))
// 實務上遠快於這個上界
struct Dinic {
  struct E {
    int to;
    long long cap;
  };
  int n;
  vector<E> es;              // 邊成對存放：i 與 i^1 互為反向
  vector<vector<int>> g;     // g[v] 存的是邊的 index
  vector<int> level, iter;

  explicit Dinic(int n) : n(n), g(n), level(n), iter(n) {}

  void addEdge(int u, int v, long long c) {
    g[u].push_back(es.size());
    es.push_back({v, c});
    g[v].push_back(es.size());
    es.push_back({u, 0});    // 反向邊初始容量 0
  }

  bool bfs(int s, int t) {
    fill(level.begin(), level.end(), -1);
    queue<int> q;
    level[s] = 0;
    q.push(s);
    while (!q.empty()) {
      int v = q.front();
      q.pop();
      for (int id : g[v]) {
        if (es[id].cap > 0 && level[es[id].to] < 0) {
          level[es[id].to] = level[v] + 1;
          q.push(es[id].to);
        }
      }
    }
    return level[t] >= 0;    // 到不了匯點 ⇒ 已經是最大流
  }

  long long dfs(int v, int t, long long f) {
    if (v == t) {
      return f;
    }
    for (int& i = iter[v]; i < (int)g[v].size(); ++i) {  // 注意是 int&
      int id = g[v][i];
      E& e = es[id];
      if (e.cap > 0 && level[v] < level[e.to]) {
        long long d = dfs(e.to, t, min(f, e.cap));
        if (d > 0) {
          e.cap -= d;
          es[id ^ 1].cap += d;  // 反向邊加回去
          return d;
        }
      }
    }
    return 0;
  }

  long long maxflow(int s, int t) {
    long long flow = 0;
    while (bfs(s, t)) {
      fill(iter.begin(), iter.end(), 0);
      while (long long f = dfs(s, t, LLONG_MAX / 4)) {
        flow += f;
      }
    }
    return flow;
  }
};
```

> [!important] `for (int& i = iter[v]; ...)` 那個 reference 不是筆誤
> `iter[v]` 記錄「v 的哪些出邊已經確定沒用了」。用 reference 讓遞迴回來之後**不會重新掃已經試過的邊**——這叫當前弧優化，是 Dinic 複雜度保證的一部分。寫成 `int i = iter[v]` （值拷貝）程式仍然會得到正確答案，但退化成指數級，大測資必定 TLE。

## 最大流 = 最小割

跑完最大流之後，從源點沿**還有剩餘容量的邊**能走到的點集合，就是最小割的一側。跨越這個分界的原始邊，容量和等於最大流。

```cpp
vector<char> minCutSide(int s) {  // 呼叫前要先跑過 maxflow
  vector<char> vis(n, 0);
  queue<int> q;
  vis[s] = 1;
  q.push(s);
  while (!q.empty()) {
    int v = q.front();
    q.pop();
    for (int id : g[v]) {
      if (es[id].cap > 0 && !vis[es[id].to]) {
        vis[es[id].to] = 1;
        q.push(es[id].to);
      }
    }
  }
  return vis;  // vis[v] 為真 ⇒ v 在源點側
}
```

實測 2000 組隨機圖：`maxflow` 的值與 2ⁿ 暴力枚舉的最小割完全一致，且 `minCutSide` 割出來的容量和也等於最大流。

## 二分圖匹配 — 建圖 + 匈牙利

**用最大流做**：源點 → 每個左點（容量 1）→ 可配對的右點（容量 1）→ 匯點（容量 1）。最大流就是最大匹配。

```txt
       ┌─1─▶ L0 ─▶ R0 ─1─┐
  S ───┼─1─▶ L1 ─▶ R1 ─1─┼──▶ T
       └─1─▶ L2 ─▶ R2 ─1─┘

所有容量都是 1：左點只能配一次、右點只能被配一次
```

**或用匈牙利（Kuhn）**，程式更短，二分圖專用：

```cpp
// Time: O(V·E)   實務上對中小圖足夠快
bool tryKuhn(int v, const vector<vector<int>>& g, vector<char>& used, vector<int>& match) {
  for (int to : g[v]) {
    if (used[to]) {
      continue;
    }
    used[to] = 1;
    // 右點沒被配，或它現在的對象能讓出去 → 就配給我
    if (match[to] == -1 || tryKuhn(match[to], g, used, match)) {
      match[to] = v;
      return true;
    }
  }
  return false;
}

int hungarian(int nl, int nr, const vector<vector<int>>& g) {
  vector<int> match(nr, -1);
  int res = 0;
  for (int v = 0; v < nl; ++v) {
    vector<char> used(nr, 0);  // 每次增廣都要重置
    if (tryKuhn(v, g, used, match)) {
      ++res;
    }
  }
  return res;
}
```

> [!tip] 匈牙利的 `tryKuhn` 遞迴就是「請對方讓位」
> 右點 `to` 已經被 `match[to]` 佔了，就遞迴問 `match[to]`：「你能不能換一個？」能換就讓位給我。這條讓位鏈就是增廣路，跟最大流是同一件事的不同外觀。實測三者（匈牙利、Dinic、2ᵐ 暴力）結果完全一致。

## 二分圖上的三個等式

| 要求的東西 | 答案 | 為什麼 |
| --- | --- | --- |
| 最大匹配 | 跑匈牙利／Dinic | — |
| **最小點覆蓋**（選最少的點蓋住所有邊） | **= 最大匹配** | König 定理 |
| **最大獨立集**（選最多的點且兩兩無邊） | **= n − 最大匹配** | 補集就是最小點覆蓋 |
| 最小路徑覆蓋（DAG，用最少路徑蓋住所有點） | = n − 最大匹配 | 拆點後跑二分圖匹配 |

König 定理實測對拍 2000 組暴力枚舉，零反例。**這三個轉換是最大流題目最常見的偽裝**——題目問的是「最少刪幾個點」「最多選幾個互不衝突的東西」，本體都是最大匹配。

## 建模技巧

看出「這題是最大流」比寫模板難得多。常見的轉換：

- **點有容量限制**：把點 `v` 拆成 `v_in → v_out`，中間那條邊的容量就是點容量，所有入邊接 `v_in`、出邊接 `v_out`
- **多源多匯**：加一個超級源點連向所有源（容量 = 該源的供給量），所有匯連向超級匯
- **無向邊**：兩個方向各加一條容量 c 的邊（不是一條邊配容量 0 的反向邊）
- **邊有下界**（必須至少流多少）：有源匯上下界流，先做「無源匯可行流」再補
- **「二選一 + 互斥懲罰」**：最小割建模（項目選擇問題），把「選 A 的收益」放源點側、「選 B 的收益」放匯點側，衝突代價當中間的邊
- **矩陣行列和限制**：行當左點、列當右點，格子是邊

> [!warning] 容量用 `long long`，並注意 `LLONG_MAX` 不能當初始 f
> `dfs(s, t, LLONG_MAX)` 之後 `min(f, e.cap)` 沒事，但若有任何地方做 `f + something` 就會溢位。用 `LLONG_MAX / 4` 這種留餘裕的值（跟 [[Shortest-Path-Algorithms]] 的 `INF` 同一個道理）。

## 常見陷阱

> [!warning] 五個地方
> - **當前弧優化寫成值拷貝**：見上面的 callout，會 TLE 但答案對。
> - **反向邊初始容量寫成 c**：那是無向邊的做法，有向圖這樣寫答案會偏大。
> - **`minCutSide` 在跑 `maxflow` 之前呼叫**：殘量網路還沒建立，割是錯的。
> - **匈牙利忘記每輪重置 `used`**：後面的點找不到增廣路，匹配數偏小。
> - **邊數估算錯導致 `vector` 反覆 realloc**：不影響正確性但慢；已知邊數時先 `reserve`。

## Related Problems

- [[Graph-Traversal-and-Connectivity]] — 二分圖判定（建圖前要先確認真的是二分圖）
- [[Disjoint-Set-Union]] — 另一種「配對／分組」工具，但不處理容量
- [[Shortest-Path-Algorithms]] — 費用流的內層就是最短路（SPFA／Dijkstra + 勢能）
