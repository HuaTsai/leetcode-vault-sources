---
tags:
  - graph
  - data-structure
  - pattern
dg-publish: true
---

## 這篇在解決什麼

「這兩個點連在一起了嗎？」這個問題，用圖走訪每次要 O(n + m)，問 q 次就 O(q(n+m))。DSU（並查集 / Union-Find）把它壓到**均攤近乎 O(1)**，代價是只能回答「連不連」，不能回答「怎麼連」——它不記路徑，只記歸屬。

它的實作短到十行，但那十行裡有**兩個各自獨立的優化**，少做一個就會從近 O(1) 退化。這篇把兩者的作用拆開量給你看，另收一個「不能做路徑壓縮」的變體。

```txt
DSU 維護一片森林，每棵樹 = 一個集合，樹根 = 集合代表

  初始              unite(1,2)   unite(3,4)   unite(2,4)

  0 1 2 3 4           1            1   3          1
                      |            |   |         /|\
                      2            2   4        2 3
                                                  |
                                                  4

find(4) 一路往上走到根 → 1        same(2,4)：兩邊 find 都得到 1 → 連通
```

## 模板：路徑壓縮 + 按大小合併

```cpp
// find / unite 均攤 O(α(n))，α 是反 Ackermann 函數，n = 10^9 時 α < 5
// Space: O(n)
struct DSU {
  vector<int> parent, sz;
  int components;  // 目前有幾個集合，很多題直接要這個

  explicit DSU(int n) : parent(n), sz(n, 1), components(n) {
    iota(parent.begin(), parent.end(), 0);  // 每個點自成一個集合
  }

  int find(int x) {
    return parent[x] == x ? x : parent[x] = find(parent[x]);  // 回傳時順手改寫 parent
  }

  bool unite(int a, int b) {
    a = find(a), b = find(b);
    if (a == b) {
      return false;  // 本來就同一集合，回傳 false 讓呼叫端能算「有幾條邊是多餘的」
    }
    if (sz[a] < sz[b]) {
      swap(a, b);  // 永遠讓小的掛到大的下面
    }
    parent[b] = a;
    sz[a] += sz[b];
    --components;
    return true;
  }

  bool same(int a, int b) { return find(a) == find(b); }
  int size(int x) { return sz[find(x)]; }
};
```

> [!tip] `unite` 回傳 bool 比回傳 void 有用得多
> `false` 代表「這兩點本來就連通」。Kruskal 靠它判斷一條邊該不該選，「找出圖中多餘的邊」「判斷加這條邊會不會成環」全都是同一個回傳值。寫成 `void` 之後這些題都要在外面再 `same()` 一次，白跑一趟 `find`。

## 兩個優化各自在做什麼

這兩件事**目標不同**：路徑壓縮讓已經走過的路變短，按大小合併讓路一開始就不會變長。都做才有 α(n)。

**路徑壓縮的效果** — 人工造出一條深度 n 的鏈，對每個節點各 `find` 一次：

| n | 無路徑壓縮 | 有路徑壓縮 | 倍數 |
| --- | --- | --- | --- |
| 5000 | 20.91 ms | 0.0210 ms | 995× |
| 10000 | 83.66 ms | 0.0419 ms | 1996× |
| 20000 | 341.88 ms | 0.0860 ms | 3978× |
| 40000 | 1344.08 ms | 0.1781 ms | 7546× |

n 每翻一倍，無壓縮的時間 **×4**（O(n²)），有壓縮的 **×2**（O(n)，因為第一次走完就把整條路攤平了）。

**按大小合併的效果** — 全程不做路徑壓縮，只看最終樹有多深：

| n | 對抗序列・不按大小 | 對抗序列・按大小 | 隨機邊・不按大小 | 隨機邊・按大小 | log₂(n) |
| --- | --- | --- | --- | --- | --- |
| 1000 | 999 | 1 | 197 | 5 | 10.0 |
| 10000 | 9999 | 1 | 1163 | 6 | 13.3 |
| 100000 | 99999 | 1 | 9810 | 7 | 16.6 |
| 1000000 | 999999 | 1 | 95384 | 8 | 19.9 |

> [!important] 按大小合併的深度上界 log₂(n) 是硬保證，不是平均情況
> 一個節點的深度每加 1，它所在的樹至少要**大一倍**（因為只有小樹掛到大樹下時深度才會增加）。n 個點最多翻倍 log₂(n) 次，所以深度必 ≤ log₂(n)——上表按大小那兩欄從沒超過。
>
> 反過來，不按大小連**隨機**邊都會退化到約 0.1n（10⁶ 時深度 95384），不需要對抗性輸入。這也是為什麼「反正有路徑壓縮就好」是錯的：壓縮只在 `find` 被呼叫**之後**才生效，一連串 `unite` 中間的每一次 `find` 都得先付那個深度的代價。

## 變體：可撤銷 DSU — 不能做路徑壓縮

離線分治、回溯搜尋、「動態圖連通性」這類題目需要把合併**收回來**。路徑壓縮會改寫沿路所有節點的 `parent`，改了什麼沒有記錄，所以**做了壓縮就撤不回去**。這個版本只保留按大小合併（深度 O(log n) 已經夠用），並用一個 stack 記下每次合併動了誰。

```cpp
// unite / find: O(log n)（沒有壓縮，只靠按大小合併壓住深度）
// rollback: 每撤銷一次合併 O(1)
struct RollbackDSU {
  vector<int> parent, sz;
  vector<pair<int, int>> history;  // (被掛上去的那個根, 它原本的 size)
  int components;

  explicit RollbackDSU(int n) : parent(n), sz(n, 1), components(n) {
    iota(parent.begin(), parent.end(), 0);
  }

  int find(int x) {
    while (parent[x] != x) {
      x = parent[x];  // 絕對不能寫成 parent[x] = parent[parent[x]]
    }
    return x;
  }

  bool unite(int a, int b) {
    a = find(a), b = find(b);
    if (a == b) {
      return false;
    }
    if (sz[a] < sz[b]) {
      swap(a, b);
    }
    history.push_back({b, sz[b]});
    parent[b] = a;
    sz[a] += sz[b];
    --components;
    return true;
  }

  int snapshot() const { return history.size(); }  // 記下目前狀態

  void rollback(int t) {  // 撤銷到某個 snapshot
    while ((int)history.size() > t) {
      auto [b, oldSize] = history.back();
      history.pop_back();
      sz[parent[b]] -= oldSize;
      parent[b] = b;
      ++components;
    }
  }
};
```

用法是「存檔 → 亂搞 → 讀檔」：

```cpp
int snap = dsu.snapshot();
dsu.unite(a, b);
// ... 這條分支的搜尋 ...
dsu.rollback(snap);  // 完全回到 unite 之前
```

## 常見陷阱

> [!warning] 五個會讓 DSU 悄悄變慢或算錯的地方
> - **`find` 寫成 `return parent[x] == x ? x : find(parent[x])`**——少了 `parent[x] =` 那個賦值就沒有壓縮，退化成上面表格的左欄。這個手滑不會報錯，只會 TLE。
> - **`unite` 裡忘了先 `find`**，直接 `parent[b] = a`：把兩個**非根**節點接起來，整個結構就壞了，而且錯得很晚才發現。
> - **深鏈上用遞迴版 `find` 會爆堆疊**。有按大小合併時深度 ≤ log₂(n)，遞迴很安全；但如果你的版本沒有按大小合併（或用了 `RollbackDSU` 的介面之外的東西），n = 10⁶ 的鏈會直接 segfault。要保險就用 two-pass 迭代版：先走到根，再走第二趟把沿路全部指向根。
> - **`size(x)` 寫成 `sz[x]`**：`sz` 只在**根**上有意義，非根節點的 `sz` 是它被合併時的舊值，是垃圾。一定要 `sz[find(x)]`。
> - **可撤銷版做了路徑壓縮**：撤銷不回來，而且同樣不會報錯——只會在某個測資上得到錯誤答案。

## 典型用法

- **Kruskal 最小生成樹**：邊按權重排序，`unite` 回傳 `true` 就選這條邊
- **判斷成環**：加邊時 `unite` 回傳 `false` ⇒ 這條邊造成環
- **連通塊計數／最大連通塊**：直接讀 `components` 和 `size(x)`
- **離線逆向處理**：「刪邊」問題把操作反過來變成「加邊」，因為 DSU 只能合併不能分裂
- **帶權 DSU**：`parent` 之外再存「到父節點的關係」（差值、模某數的偏移），可以回答「a 和 b 差多少」——注意壓縮時要同步累加權重
- **二分圖判定**：開 `2n` 個點，`i` 和 `i+n` 代表「i 在左邊／右邊」，每條邊 `unite(u, v+n)` 和 `unite(v, u+n)`；若某刻 `same(i, i+n)` 就不是二分圖

## Related Problems

- [[0547-Number-of-Provinces]] — 最直接的連通塊計數
- [[0684-Redundant-Connection]] — `unite` 回傳 `false` 的那條邊就是答案
- [[0721-Accounts-Merge]] — 把字串映射成整數後的標準 DSU
- [[1584-Min-Cost-to-Connect-All-Points]] — Kruskal + DSU
- [[Minimum-Spanning-Tree]] — Kruskal 的完整寫法與 Prim 的取捨
