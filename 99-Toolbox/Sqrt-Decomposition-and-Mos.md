---
tags:
  - data-structure
  - range-query
  - pattern
dg-publish: true
---

## 這篇在解決什麼

[[Segment-Tree]] 和 [[Fenwick-Tree]] 很強，但它們要求操作能**快速合併**（結合律 + 可用父節點的值代表整段）。有些操作沒有這種結構：

- 「區間內有幾個**不同**的數」——兩段的答案沒辦法合併成整段的答案
- 「區間眾數」、「區間第 k 小（線上）」——同樣不可合併

這時候退一步用 **O(√n)** 的方法反而可行。分塊（sqrt decomposition）是通用的「暴力 + 預存」折衷，Mo's algorithm 則是它在**離線查詢**上的特化。

對拍驗過：分塊的區間加／區間和 30000 組、Mo's 的區間不同數 7602 組，全過。

## 分塊 — 把陣列切成 √n 塊

```txt
n = 16，塊大小 B = 4

  [ 0  1  2  3 ][ 4  5  6  7 ][ 8  9 10 11 ][12 13 14 15 ]
   └── 塊 0 ──┘ └── 塊 1 ──┘ └── 塊 2 ──┘ └── 塊 3 ──┘
        每塊預存一個 sum，整塊操作只要動這個 sum

查詢 [2, 13] 拆成三段：
  [2,3]    左邊零頭 → 逐個暴力，最多 B 個
  塊1、塊2  中間整塊 → 直接讀預存的 sum，最多 n/B 塊
  [12,13]  右邊零頭 → 逐個暴力，最多 B 個

總成本 O(B + n/B)，在 B = √n 時最小 ⇒ O(√n)
```

```cpp
// build: O(n)   rangeAdd / rangeSum: O(sqrt(n))
struct SqrtDecomp {
  int n, B, nb;
  vector<long long> a, sum, lazy;  // lazy[b] = 整塊 b 被加了多少

  explicit SqrtDecomp(const vector<long long>& v) : n(v.size()), a(v) {
    B = max(1, (int)sqrt((double)n));
    nb = (n + B - 1) / B;
    sum.assign(nb, 0);
    lazy.assign(nb, 0);
    for (int i = 0; i < n; ++i) {
      sum[i / B] += a[i];
    }
  }

  void rangeAdd(int l, int r, long long v) {
    int bl = l / B, br = r / B;
    if (bl == br) {                       // 同一塊：整段都是零頭
      for (int i = l; i <= r; ++i) {
        a[i] += v;
      }
      sum[bl] += v * (r - l + 1);
      return;
    }
    for (int i = l; i < (bl + 1) * B; ++i) {   // 左零頭
      a[i] += v;
    }
    sum[bl] += v * ((bl + 1) * B - l);
    for (int b = bl + 1; b < br; ++b) {        // 中間整塊：只動 lazy
      lazy[b] += v;
      sum[b] += v * B;
    }
    for (int i = br * B; i <= r; ++i) {        // 右零頭
      a[i] += v;
    }
    sum[br] += v * (r - br * B + 1);
  }

  long long rangeSum(int l, int r) {
    int bl = l / B, br = r / B;
    long long res = 0;
    if (bl == br) {
      for (int i = l; i <= r; ++i) {
        res += a[i] + lazy[bl];  // 零頭要補上整塊的 lazy
      }
      return res;
    }
    for (int i = l; i < (bl + 1) * B; ++i) {
      res += a[i] + lazy[bl];
    }
    for (int b = bl + 1; b < br; ++b) {
      res += sum[b];             // 整塊：sum 已經含 lazy
    }
    for (int i = br * B; i <= r; ++i) {
      res += a[i] + lazy[br];
    }
    return res;
  }
};
```

> [!important] 零頭讀值時要補 `lazy[block]`，整塊讀 `sum` 時不用
> `lazy[b]` 是「這一塊每個元素都欠 lazy[b]」，而 `a[i]` 沒有被更新——所以逐個讀時必須補。`sum[b]` 在打標記時就已經加了 `v * B`，是完整的值。搞混就會少加或重複加。這跟 [[Segment-Tree]] 的 lazy「欠子孫不欠自己」是類似的記帳問題，但方向相反。

> [!tip] 分塊的真正價值是「什麼都能做」
> 區間和用線段樹更快。分塊值錢的地方在於**它不要求可合併性**：每塊裡面可以存一個排序好的 vector（回答「區間內大於 x 的有幾個」）、一個計數表（區間眾數）、任何你想得到的東西。線段樹做不到的，分塊往往還撐得住——**代價是 √n 而不是 log n**。

## Mo's algorithm — 離線查詢重排

所有查詢**事先都知道**（離線）時，可以重排它們的處理順序，讓 `[l, r]` 窗口的移動總量最小。維護一個「加入一個元素 / 刪除一個元素」的增量結構就行——**不需要任何合併能力**。

```txt
排序規則：先按 l 所在的塊，同塊內按 r

  l 的移動：同一塊內 l 最多跑 B 步，共 Q 次 ⇒ O(QB)
  r 的移動：同一塊內 r 單調遞增最多 n 步，共 n/B 塊 ⇒ O(n²/B)

  B = n/√Q 時最佳；實務上取 B = √n，總複雜度 O((n + Q)√n)
```

```cpp
// Time: O((n + Q) sqrt(n))   前提：查詢可離線，且能 O(1) 增刪單一元素
struct MoQuery {
  int l, r, idx;  // idx 用來把答案放回原本的順序
};

vector<int> mosDistinct(const vector<int>& a, vector<MoQuery> qs, int maxVal) {
  int n = a.size(), B = max(1, (int)sqrt((double)n));

  sort(qs.begin(), qs.end(), [&](const MoQuery& x, const MoQuery& y) {
    int bx = x.l / B, by = y.l / B;
    if (bx != by) {
      return bx < by;
    }
    return (bx & 1) ? x.r > y.r : x.r < y.r;  // 奇偶排序，見下
  });

  vector<int> cnt(maxVal + 1, 0), res(qs.size());
  int curL = 0, curR = -1, distinct = 0;

  auto add = [&](int i) {
    if (cnt[a[i]]++ == 0) {
      ++distinct;  // 從 0 變 1：多一種
    }
  };
  auto del = [&](int i) {
    if (--cnt[a[i]] == 0) {
      --distinct;  // 從 1 變 0：少一種
    }
  };

  for (const auto& q : qs) {
    while (curR < q.r) add(++curR);   // 四個 while 的順序：先擴張再收縮
    while (curL > q.l) add(--curL);
    while (curR > q.r) del(curR--);
    while (curL < q.l) del(curL++);
    res[q.idx] = distinct;
  }
  return res;
}
```

> [!warning] 四個 `while` 的順序不能亂
> **必須先擴張（`add`）再收縮（`del`）**。若先收縮，窗口可能在中途變成 `l > r + 1` 的非法狀態，`del` 會刪到不在窗口裡的元素，計數就爛了。記法是「寧可先變大」。

### 排序到底省了多少——實測

n = 10⁵、Q = 10⁵ 隨機查詢，統計 `l`／`r` 指標的移動總次數：

| 處理順序 | 指標移動總次數 |
| --- | --- |
| 完全不排序（照輸入順序） | 5,323,037,021 |
| 按 `(l 的塊, r)` 排序 | 41,938,385 |
| **再加上奇偶排序** | **26,405,373** |

排序本身帶來 **127 倍**改善，奇偶排序再省掉 **37%**。

> [!tip] 奇偶排序：偶數塊 `r` 遞增、奇數塊 `r` 遞減
> 不做這個優化的話，每換一個塊，`r` 都要從最右邊「跳回」最左邊重新往右掃，那一次跳躍就是 O(n)。奇偶交替讓 `r` 像**來回掃描**一樣走完就掉頭，省掉所有跳回的成本。程式上只是排序比較函式裡多一個 `(bx & 1) ?`，收益卻是三分之一——CP 裡少數「一行換 1.5 倍」的優化。

## 怎麼選

| 情境 | 用什麼 |
| --- | --- |
| 區間和／max，要更新 | [[Segment-Tree]] 或 [[Fenwick-Tree]]，O(log n) |
| 靜態區間 min／max | [[Sparse-Table-RMQ]]，O(1) |
| **不可合併**的查詢，可離線 | **Mo's**，O((n+Q)√n) |
| **不可合併**的查詢，必須線上 | **分塊**，O(√n)；或可持久化資料結構 |
| 帶修改的區間不同數 | 帶修莫隊（多一維時間），O(n^(5/3)) |
| 樹上路徑的類似查詢 | 樹上莫隊（先做歐拉序） |

> [!note] √n 演算法的定位是「當 log n 做不到時」
> n = 2×10⁵ 時 √n ≈ 450、log n ≈ 18，差 25 倍。所以只要線段樹做得到就別用分塊。它們的價值在於**線段樹做不到的那些查詢**——這時候能過就是勝利。

## 常見陷阱

> [!warning] 五個地方
> - **Mo's 的四個 `while` 順序寫反**：窗口進入非法狀態，見上面的 callout。
> - **忘記用 `idx` 還原順序**：答案順序全錯。
> - **`add`／`del` 不是 O(1)**：整個複雜度分析就垮了。用 `set` 維護會變成 O(√n log n)。
> - **分塊零頭讀值忘了補 lazy**：少加。
> - **塊大小寫死成某個常數**：n 很小時 `B` 可能算出 0，`i / B` 直接除以零。記得 `max(1, ...)`。

## Related Problems

- [[0307-Range-Sum-Query-Mutable]] — 分塊也能解，但線段樹更適合
- [[1157-Online-Majority-Element-In-Subarray]] — 區間眾數，分塊的典型場景
- [[Segment-Tree]] — 可合併查詢的首選
- [[Sparse-Table-RMQ]] — 靜態不可更新時的 O(1)
