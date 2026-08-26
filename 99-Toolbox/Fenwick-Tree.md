---
tags:
  - data-structure
  - range-query
  - pattern
dg-publish: true
---

## 這篇在解決什麼

一個陣列，要一邊改值一邊問區間和。兩個極端都很差：

- **裸陣列**：改值 O(1)，求和 O(n)
- **前綴和陣列**：求和 O(1)，但改一個值要重算後面全部，O(n)

Fenwick Tree（樹狀陣列 / BIT）讓兩邊都是 **O(log n)**，而且實作只有兩行迴圈、常數比 Segment Tree 小得多。代價是它只能處理**可減的運算**（和、xor），最大值這種不能減的做不了——那是 Segment Tree 的地盤。

這篇收四種用法（單點加、區間加單點查、區間加區間和、樹狀陣列上二分）與 O(n) 建樹，全部對拍暴力陣列驗過。

## 核心：`i & -i` 把陣列切成 2 的冪次段

`lowbit(i) = i & -i` 取出 `i` 最低位的那個 1。`t[i]` 負責的區間定義為 **`(i - lowbit(i), i]`**——長度剛好是 `lowbit(i)`：

```txt
 i   二進位  lowbit  t[i] 覆蓋的區間
 1    0001     1     (0,1]  = a[1]
 2    0010     2     (0,2]  = a[1..2]
 3    0011     1     (2,3]  = a[3]
 4    0100     4     (0,4]  = a[1..4]
 5    0101     1     (4,5]  = a[5]
 6    0110     2     (4,6]  = a[5..6]
 7    0111     1     (6,7]  = a[7]
 8    1000     8     (0,8]  = a[1..8]

               8
        ┌──────┴──────┬──────┐
        4             6      7
     ┌──┴──┐          │
     2     3          5
     │
     1

prefix(7) = t[7] + t[6] + t[4]     7 → 6 → 4 → 0，每次減掉 lowbit
add(3, v) 要更新 t[3], t[4], t[8]  3 → 4 → 8，每次加上 lowbit
```

兩條迴圈方向相反：**查詢往下減 lowbit**（把區間拼起來），**更新往上加 lowbit**（把包含我的區間都改到）。兩者都最多走 log₂(n) 步，因為每步至少讓最低位的 1 往左移一格。

> [!important] 為什麼內部一定要 1-indexed
> `lowbit(0) = 0`，更新迴圈 `i += i & -i` 碰到 0 會原地打轉、查詢迴圈 `i -= i & -i` 永遠不終止。**0 是這個結構的黑洞**。標準做法是內部從 1 開始、對外仍暴露 0-indexed 介面（下面模板在 `add`／`prefix` 開頭 `++i`），這樣題目給你 0-indexed 陣列時不必在呼叫端到處 +1。

## 模板一：單點加 + 前綴和

```cpp
// add / prefix: O(log n)   建樹: O(n)   Space: O(n)
struct Fenwick {
  int n;
  vector<long long> t;  // 用 long long——區間和很容易溢位 int

  explicit Fenwick(int n) : n(n), t(n + 1, 0) {}

  explicit Fenwick(const vector<long long>& a) : n(a.size()), t(a.size() + 1, 0) {
    for (int i = 0; i < n; ++i) {  // O(n) 建樹，不是 n 次 O(log n) 的 add
      t[i + 1] += a[i];
      int j = i + 1 + ((i + 1) & -(i + 1));  // 把自己的值往上傳給父節點
      if (j <= n) {
        t[j] += t[i + 1];
      }
    }
  }

  void add(int i, long long v) {
    for (++i; i <= n; i += i & -i) {  // 往上：把所有覆蓋 i 的區間都加上 v
      t[i] += v;
    }
  }

  long long prefix(int i) {  // a[0..i] 的和
    long long s = 0;
    for (++i; i > 0; i -= i & -i) {  // 往下：把區間一段一段拼起來
      s += t[i];
    }
    return s;
  }

  long long range(int l, int r) { return prefix(r) - (l ? prefix(l - 1) : 0); }
};
```

## 模板二：區間加 + 單點查 — 存差分就好

想「區間加、單點查」時，不要在 Fenwick 裡硬做——**改成對差分陣列做單點加**。令 `d[i] = a[i] - a[i-1]`，那麼 `a[i] = d[0] + d[1] + … + d[i]`，剛好是模板一的 `prefix(i)`。區間 `[l, r]` 整段 +v，在差分上只動兩個位置：

```txt
a:  3  3  3  3  3        對 [1,3] 加 5
d:  3  0  0  0  0   →    d[1] += 5, d[4] -= 5
                         d:  3  5  0  0 -5
                         prefix: 3  8  8  8  3   ← 正是加完的 a
```

```cpp
// rangeAdd / at: O(log n)
struct FenwickRangeAdd {
  Fenwick f;
  explicit FenwickRangeAdd(int n) : f(n) {}

  void rangeAdd(int l, int r, long long v) {
    f.add(l, v);
    if (r + 1 < f.n) {
      f.add(r + 1, -v);  // 出了區間就把加上去的收回來
    }
  }

  long long at(int i) { return f.prefix(i); }  // 差分的前綴和 = 原值
};
```

## 模板三：區間加 + 區間和 — 兩棵樹

沿用差分，但這次要對 `a` 求**前綴和的前綴和**。展開一次就知道要存什麼：

```txt
prefix(i) = Σ(j=0..i) a[j] = Σ(j=0..i) Σ(k=0..j) d[k]

每個 d[k] 被算了 (i - k + 1) 次，所以

prefix(i) = Σ(k=0..i) d[k] · (i - k + 1)
          = (i+1) · Σ(k=0..i) d[k]  −  Σ(k=0..i) k · d[k]
             └──── 第一棵樹存 d[k] ──┘   └─ 第二棵樹存 k·d[k] ─┘
```

```cpp
// rangeAdd / range: O(log n)，常數約為單棵的兩倍
struct FenwickRangeSum {
  int n;
  Fenwick b1, b2;  // b1 存 d[k]，b2 存 k * d[k]
  explicit FenwickRangeSum(int n) : n(n), b1(n), b2(n) {}

  void rangeAdd(int l, int r, long long v) {
    b1.add(l, v);
    b2.add(l, v * l);
    if (r + 1 < n) {
      b1.add(r + 1, -v);
      b2.add(r + 1, -v * (r + 1));
    }
  }

  long long prefix(int i) { return b1.prefix(i) * (i + 1) - b2.prefix(i); }
  long long range(int l, int r) { return prefix(r) - (l ? prefix(l - 1) : 0); }
};
```

## 模板四：樹狀陣列上二分 — O(log n) 找第 k 小

已經有 `prefix` 了，「找最小的 `i` 使 `prefix(i) >= target`」可以外面套二分變成 O(log²n)。但 Fenwick 的結構本身就是倍增的，**直接在樹上走一次就好**，省掉一個 log：

```cpp
// Time: O(log n)  要求：所有值非負（否則 prefix 不單調，二分無意義）
int lowerBound(long long target) {
  int pos = 0;
  long long cur = 0;
  for (int pw = 1 << (31 - __builtin_clz(max(n, 1))); pw > 0; pw >>= 1) {
    if (pos + pw <= n && cur + t[pos + pw] < target) {
      pos += pw;      // 這一步跨過去仍然不夠，安全，跳
      cur += t[pos];
    }
  }
  return pos;  // 0-indexed；target 超過總和時回傳 n
}
```

> [!tip] 這個「不夠才跳」的貪心，跟 [[Binary-Lifting-LCA]] 階段二是同一招
> 兩者都是從大步試到小步、**只在確定安全時才跳**，最後停在「差一步」的位置。倍增祖先表那邊的條件是「祖先不同才跳」，這裡是「累積和還不夠才跳」——同一個骨架換個判斷式。
>
> 典型用途：把值域當 index、每個值出現次數當權重，`lowerBound(k)` 就是**第 k 小的值**。這是「動態第 k 小」的標準解。

## 常見陷阱

> [!warning] 五個踩了會錯或會慢的地方
> - **忘記 `++i`（或忘記內部 1-indexed）**：`add(0, v)` 會讓更新迴圈卡在 `i = 0` 無限打轉。
> - **用 `int` 存 `t`**：n = 2×10⁵、每個值 10⁹ 時區間和是 10¹⁴，必爆。`t` 一律 `long long`。
> - **`add` 當成賦值用**：`add(i, v)` 是**加上** v，不是設成 v。要設值得先 `add(i, v - 現值)`，而 Fenwick 沒有 O(1) 讀單值的方法（要 `range(i, i)`）。
> - **拿 Fenwick 做 max**：`prefix` 靠減法拼區間，最大值不可減。硬要做只能支援「前綴 max + 單點改大」這種受限操作，一般情況請用 Segment Tree。
> - **建樹用 n 次 `add`**：那是 O(n log n)。模板一的建構子是 O(n)，n = 10⁶ 時差很有感。

## Fenwick 還是 Segment Tree？

| | Fenwick | Segment Tree |
| --- | --- | --- |
| 程式長度 | 兩行迴圈 | 遞迴四個函式起跳 |
| 常數 | 小 | 約 2–4 倍 |
| 記憶體 | `n+1` | `4n`（遞迴版）／`2n`（迭代版） |
| 支援的運算 | 只有可減的（和、xor） | 任何可結合的（max、min、gcd、矩陣乘…） |
| 區間更新 | 要靠差分技巧，且只到「區間加」 | lazy propagation，區間賦值／區間加都行 |
| 二分找位置 | 內建 O(log n) | 要自己寫下降 |

**只要是「區間和 + 單點改」就用 Fenwick**，短、快、不會寫錯。需要 max／區間賦值／更複雜的合併才升級到 Segment Tree（見 [[Segment-Tree]]）。

## 典型用法

- **動態前綴和／區間和**：本體
- **逆序對計數**：值域當 index，從右往左掃，每步先 `prefix(a[i]-1)` 再 `add(a[i], 1)`
- **動態第 k 小**：`lowerBound(k)`，見模板四
- **離線區間查詢**：把查詢按右端點排序，一邊掃一邊 `add`
- **二維 Fenwick**：`t[i][j]`，兩層 lowbit 迴圈，查子矩形和

## Related Problems

- [[0307-Range-Sum-Query-Mutable]] — 模板一的原題
- [[0315-Count-of-Smaller-Numbers-After-Self]] — 值域 Fenwick 數逆序對
- [[0493-Reverse-Pairs]] — 同上但條件變成 `a[i] > 2*a[j]`
- [[Segment-Tree]] — 需要 max／區間賦值時的升級版
- [[Binary-Lifting-LCA]] — 模板四的倍增下降跟它是同一招
