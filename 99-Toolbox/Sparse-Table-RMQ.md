---
tags:
  - data-structure
  - range-query
  - pattern
dg-publish: true
---

## 這篇在解決什麼

陣列**建好之後不再改**，但要問幾十萬次「區間最小值是多少」。[[Segment-Tree]] 每次查 O(log n) 已經很好，但 Sparse Table 能做到**每次 O(1)**，代價是 O(n log n) 的預處理和記憶體，而且**完全不支援更新**。

它的限制很特別：只對**冪等運算**成立（`min`、`max`、`gcd`、`按位 and/or`——重複算一次不影響結果的那些）。`sum` 不行，而且錯得很安靜。這篇把原因和實測反例一起收，對拍暴力 50000 組驗過。

## 核心：用兩段重疊的 2^k 蓋住整個區間

`st[k][i]` = 從 `i` 開始、長度 `2^k` 這段的最小值。任何區間 `[l, r]` 都可以用**兩段長度 2^k 的區塊**蓋住，其中 `k = ⌊log₂(r-l+1)⌋`：一段貼左邊界、一段貼右邊界。

```txt
查 [2, 9]（長度 8，k = 3，2^3 = 8 剛好）
 index  0  1  2  3  4  5  6  7  8  9 10
              └──────── st[3][2] ────────┘        一段就夠

查 [2, 8]（長度 7，k = 2，2^2 = 4）
 index  0  1  2  3  4  5  6  7  8  9 10
              └─ st[2][2] ─┘                      蓋住 [2,5]
                       └─ st[2][5] ─┘             蓋住 [5,8]
                       ↑↑ 重疊了 [5,5]

兩段一定蓋得滿（因為 2 × 2^k ≥ 區間長度），但**會重疊**——
這就是為什麼運算必須冪等：重疊的部分被算了兩次。
```

## 實作

```cpp
// build: O(n log n) 時間與記憶體   query: O(1)
// 只支援不更新的靜態陣列；運算必須冪等（min / max / gcd / and / or）
struct SparseTable {
  vector<vector<long long>> st;
  vector<int> lg;  // lg[x] = floor(log2(x))，預先算好避免每次呼叫 log2

  explicit SparseTable(const vector<long long>& a) {
    int n = a.size(), K = 1;
    while ((1 << K) <= n) {
      ++K;
    }
    lg.assign(n + 1, 0);
    for (int i = 2; i <= n; ++i) {
      lg[i] = lg[i / 2] + 1;
    }
    st.assign(K, vector<long long>(n));
    st[0] = a;
    for (int k = 1; k < K; ++k) {
      for (int i = 0; i + (1 << k) <= n; ++i) {
        // 長度 2^k 的段 = 兩個長度 2^(k-1) 的段合併
        st[k][i] = min(st[k - 1][i], st[k - 1][i + (1 << (k - 1))]);
      }
    }
  }

  long long query(int l, int r) {  // 閉區間 [l, r]
    int k = lg[r - l + 1];
    return min(st[k][l], st[k][r - (1 << k) + 1]);
  }
};
```

> [!important] 把 `min` 換成 `sum` 會錯，而且不會報錯
> 實測：`a = {1, 2, 3, 4, 5}`，查 `[0, 4]`（長度 5，k = 2）。兩段是 `st[2][0]`（= 1+2+3+4 = 10）和 `st[2][1]`（= 2+3+4+5 = 14），相加得 **24**，正解是 **15**——重疊的 `[1,3]` 被算了兩次。
>
> `min` 沒事是因為 `min(x, x) = x`；`sum` 的 `x + x ≠ x`。要區間和就用 [[Fenwick-Tree]]（前綴和相減即可，靜態的話連 Fenwick 都不用）。

> [!tip] `lg` 表要預先算，不要在 `query` 裡呼叫 `log2()`
> `std::log2` 是浮點運算，比查表慢一個數量級，而且**邊界會因為浮點誤差出錯**（`log2(8)` 可能算出 2.9999…，取整變成 2）。遞推 `lg[i] = lg[i/2] + 1` 是 O(n) 且精確。另一個選擇是 `31 - __builtin_clz(x)`，同樣精確且不用表。

## 什麼時候用它

| 需求 | 選擇 |
| --- | --- |
| 靜態陣列 + 大量 min／max／gcd 查詢 | **Sparse Table**，O(1) 查詢 |
| 需要更新 | [[Segment-Tree]]（Sparse Table 完全做不到） |
| 區間和 | [[Fenwick-Tree]]，或靜態就直接前綴和 |
| n 很大、記憶體吃緊 | Segment Tree（O(n) 記憶體 vs O(n log n)） |
| 查詢次數少 | Segment Tree 就好，省掉建表時間 |

分界線大概是：**查詢次數遠多於 n 時，Sparse Table 的 O(1) 才賺得回 O(n log n) 的建表成本**。n = 2×10⁵、查詢 10⁶ 次就很值得；查詢只有 10³ 次就不必。

## 應用：Euler tour + Sparse Table 求 LCA

[[Binary-Lifting-LCA]] 的選型表裡提過「O(n log n) 預處理 / O(1) 查詢」的那一格，就是這個組合：

```txt
1. 對樹做一次 DFS，記錄「歐拉序」——每次進入或回到一個節點就記一筆
   （長度 2n-1），同時記錄每個節點的深度
2. 兩點 u、v 的 LCA = 歐拉序上 [first[u], first[v]] 這段裡深度最小的那個節點
3. 「區間最小值」正是 Sparse Table 的本體 → O(1)
```

它比倍增快（查詢 O(1) vs O(log n)），但**只能求 LCA**——倍增表還能順便回答「k 級祖先」和「路徑上的最大邊權」，那些是 Sparse Table 做不到的。所以競程裡倍增更常見。

## 常見陷阱

> [!warning] 四個地方
> - **拿去做 sum**：見上面的反例。
> - **`K` 開太小**：`while ((1 << K) <= n) ++K;` 要確保 `2^(K-1) ≥ n`，不然大區間查不到。
> - **在 `query` 裡呼叫 `log2()`**：慢且有浮點誤差。
> - **建完之後想改值**：改一個位置要重建整張表，O(n log n)。要更新就換 Segment Tree，沒有折衷方案。

## Related Problems

- [[0239-Sliding-Window-Maximum]] — 定長窗的 max，單調隊列 O(n) 更好，但 Sparse Table 也能解
- [[1521-Find-a-Value-of-a-Mysterious-Function-Closest-to-Target]] — 區間 and，冪等運算的典型
- [[Segment-Tree]] — 需要更新時的替代
- [[Fenwick-Tree]] — 區間和該用的東西
- [[Binary-Lifting-LCA]] — Euler tour + RMQ 求 LCA 的對照組
