---
tags:
  - graph
  - pattern
dg-publish: true
---

## 這篇在解決什麼

有向圖裡「互相到得了」的最大點集叫**強連通分量**（SCC）。把每個 SCC 縮成一個點之後，整張圖**必定變成 DAG**——這是 SCC 真正的價值：任何在 DAG 上好做的事（拓撲排序、DP、最長路），在一般有向圖上先縮點就能做。

兩個標準演算法都是 O(n + m)，另外 2-SAT 也是 SCC 的直接應用。這篇三個都收，對拍驗過：SCC 對照「互相可達」的定義（Floyd 傳遞閉包）3000 組、2-SAT 對照 2ⁿ 暴力枚舉 5000 組，並額外驗證構造出來的賦值真的滿足所有子句。

```txt
     0 ──▶ 1 ──▶ 3 ──▶ 4
     ▲     │     ▲     │
     └──── 2 ◀───┘     └──▶ 5

SCC：{0,1,2}（互相到得了）、{3,4}、{5}

縮點後：  [0,1,2] ──▶ [3,4] ──▶ [5]      一定是 DAG
```

## Tarjan — 一次 DFS，靠 low-link

```txt
disc[v]：v 第一次被訪問的時間戳
low[v] ：v 的子樹裡（含 v 自己），沿著樹邊往下、最後最多走一條
         「回到還在堆疊上的點」的邊，所能到達的最小 disc

low[v] == disc[v]  ⇒  v 是它所在 SCC 的根
                      此刻堆疊上從頂端到 v 的那一段，就是一個完整的 SCC
```

```cpp
// Time: O(n + m)   Space: O(n)
struct TarjanSCC {
  int n, timer = 0, sccCount = 0;
  vector<vector<int>> adj;
  vector<int> disc, low, comp, stk;
  vector<char> onStack;

  explicit TarjanSCC(int n)
      : n(n), adj(n), disc(n, -1), low(n), comp(n, -1), onStack(n, 0) {}

  void addEdge(int u, int v) { adj[u].push_back(v); }

  void dfs(int v) {
    disc[v] = low[v] = timer++;
    stk.push_back(v);
    onStack[v] = 1;
    for (int to : adj[v]) {
      if (disc[to] == -1) {
        dfs(to);
        low[v] = min(low[v], low[to]);   // 樹邊：吸收子樹的 low
      } else if (onStack[to]) {
        low[v] = min(low[v], disc[to]);  // 回邊：用 disc，且只認堆疊上的點
      }
    }
    if (low[v] == disc[v]) {
      while (true) {
        int u = stk.back();
        stk.pop_back();
        onStack[u] = 0;
        comp[u] = sccCount;
        if (u == v) {
          break;
        }
      }
      ++sccCount;
    }
  }

  void build() {
    for (int i = 0; i < n; ++i) {
      if (disc[i] == -1) {
        dfs(i);
      }
    }
  }
};
```

> [!important] 那兩行 `low` 更新的差別是整個演算法的關鍵
> - **樹邊**用 `low[to]`：子樹能爬多高，我就能爬多高。
> - **回邊**用 `disc[to]` 而不是 `low[to]`，而且**只在 `onStack[to]` 時才算**。
>
> 若對已經完成的（不在堆疊上的）節點也更新 `low`，就會把「已經定案的別的 SCC」錯誤地併進來。`onStack` 這個判斷就是在區分「回邊／橫叉邊指向的是我的祖先，還是另一個已完成的 SCC」。用 `low[to]` 而非 `disc[to]` 則會讓 `low` 傳得太遠，同樣把不該合併的併在一起。

> [!tip] Tarjan 的 `comp` 編號是**逆拓撲序**
> 先完成的 SCC 拿到較小的編號，而先完成代表它在縮點後的 DAG 上位於**後方**。所以對任何一條跨 SCC 的邊 `u → v`，必有 `comp[u] > comp[v]`（實測 3000 組零反例）。
>
> 這是免費的拓撲序：**要縮點後照拓撲序 DP，直接按 `comp` 由大到小跑就好**，不必再排一次。2-SAT 的取值規則也是靠這個性質。

## Kosaraju — 兩次 DFS，比較好記

第一次 DFS 記錄完成順序，第二次在**反圖**上按完成順序的反序走，每次能走到的就是一個 SCC。

```cpp
// Time: O(n + m)   Space: O(n + m)（要多存一份反圖）
struct Kosaraju {
  int n, sccCount = 0;
  vector<vector<int>> adj, radj;
  vector<int> order, comp;
  vector<char> vis;

  explicit Kosaraju(int n) : n(n), adj(n), radj(n), comp(n, -1), vis(n, 0) {}

  void addEdge(int u, int v) {
    adj[u].push_back(v);
    radj[v].push_back(u);  // 同時建反圖
  }

  void dfs1(int v) {
    vis[v] = 1;
    for (int to : adj[v]) {
      if (!vis[to]) {
        dfs1(to);
      }
    }
    order.push_back(v);  // 後序：完成順序
  }

  void dfs2(int v, int c) {
    comp[v] = c;
    for (int to : radj[v]) {
      if (comp[to] == -1) {
        dfs2(to, c);
      }
    }
  }

  void build() {
    for (int i = 0; i < n; ++i) {
      if (!vis[i]) {
        dfs1(i);
      }
    }
    for (int i = n - 1; i >= 0; --i) {  // 完成順序的反序
      if (comp[order[i]] == -1) {
        dfs2(order[i], sccCount++);
      }
    }
  }
};
```

> [!note] 選哪個
> Tarjan 只走一次、不用建反圖，常數較小，且 `comp` 直接是逆拓撲序——競程預設用它。Kosaraju 的優點是**好記好推導**（「反圖上按完成序反著走」一句話就講完），臨場忘記 Tarjan 的 `low` 更新細節時是可靠的備案。兩者在 3000 組隨機圖上結果完全一致（SCC 個數與分組都相同），只是編號順序不同。

## 2-SAT — SCC 的招牌應用

每個布林變數 `x` 拆成兩個節點：`2x`（x 為真）和 `2x+1`（x 為假）。子句 `(a ∨ b)` 等價於兩條蘊含邊：

```txt
(a ∨ b)  ⟺  ¬a → b   且   ¬b → a
             （a 不成立時 b 就必須成立，反之亦然）

可滿足 ⟺ 對每個變數 i，節點 2i 和 2i+1 不在同一個 SCC
          （同一個 SCC 代表 x → ¬x 且 ¬x → x，矛盾）

取值：x_i = true ⟺ comp[2i] < comp[2i+1]
      直覺是「取拓撲序在後的那個」，因為蘊含邊只往拓撲序後方走，
      選後面的不會逼出矛盾。Tarjan 編號是逆拓撲，所以是「編號小的」。
```

```cpp
// Time: O(n + m)
struct TwoSat {
  int n;  // 變數個數
  TarjanSCC scc;

  explicit TwoSat(int n) : n(n), scc(2 * n) {}

  int lit(int x, bool val) { return 2 * x + (val ? 0 : 1); }

  // 加入子句 (x == xv ∨ y == yv)
  void addClause(int x, bool xv, int y, bool yv) {
    scc.addEdge(lit(x, !xv), lit(y, yv));  // ¬(x==xv) → (y==yv)
    scc.addEdge(lit(y, !yv), lit(x, xv));
  }

  bool solve(vector<char>& assign) {
    scc.build();
    assign.assign(n, 0);
    for (int i = 0; i < n; ++i) {
      if (scc.comp[2 * i] == scc.comp[2 * i + 1]) {
        return false;  // x 與 ¬x 在同一個 SCC ⇒ 無解
      }
      assign[i] = scc.comp[2 * i] < scc.comp[2 * i + 1];
    }
    return true;
  }
};
```

> [!warning] 每個子句一定要加**兩條**邊
> `(a ∨ b)` 產生 `¬a → b` 和 `¬b → a`，少加一條會讓蘊含圖不對稱，判斷可滿足性可能還是對的，但**構造出來的解會是錯的**。實測 5000 組隨機子句集：可滿足性判斷與 2ⁿ 暴力完全一致，且構造出的賦值全部通過逐子句檢查——那個檢查值得在自己寫完之後也跑一次。
>
> 強制條件「x 必須為真」寫成 `addClause(x, true, x, true)`（自己跟自己或），會產生 `¬x → x`，效果正確。

## 常見陷阱

> [!warning] 五個地方
> - **Tarjan 回邊誤用 `low[to]`**：`low` 傳太遠，不同 SCC 被併在一起。
> - **忘記 `onStack` 判斷**：橫叉邊指向已完成的 SCC 也被算進來，同樣錯併。
> - **遞迴深度**：n = 10⁵ 的鏈會爆堆疊。CSES 這種 n 到 10⁵ 的題目要改迭代版 Tarjan，或確認評測環境的堆疊夠大。
> - **2-SAT 只加一條邊**：見上面的 callout。
> - **以為縮點後還要自己排拓撲序**：Tarjan 的 `comp` 已經是逆拓撲序，白排一次。

## 典型用法

- **縮點後在 DAG 上 DP**：最長路、能到達的最多點數（CSES *Coin Collector* 就是縮點 + DAG 上最長路）
- **判斷「是否存在一個點能到達所有點」**：縮點後出度為 0 的 SCC 若恰有一個，且它的大小 = 該 SCC 點數，答案就在裡面
- **最少加幾條邊讓整張圖強連通**：縮點後 `max(入度為 0 的 SCC 數, 出度為 0 的 SCC 數)`（整張圖本來就強連通時答案 0）
- **2-SAT**：排班、塗色、「兩個選項二選一且有互斥限制」這類問題
- **必經點／橋**：Tarjan 的 low-link 換個判斷條件就變成割點與橋（無向圖版）

## Related Problems

- [[1192-Critical-Connections-in-a-Network]] — 同一套 low-link，求橋
- [[0207-Course-Schedule]] — 環偵測，SCC 的簡化版（見 [[Graph-Traversal-and-Connectivity]]）
- [[Graph-Traversal-and-Connectivity]] — 拓撲排序與三色環偵測
- [[Disjoint-Set-Union]] — 無向圖的連通性工具
