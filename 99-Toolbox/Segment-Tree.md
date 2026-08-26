---
tags:
  - data-structure
  - range-query
  - pattern
dg-publish: true
---

## 這篇在解決什麼

[[Fenwick-Tree]] 已經解決「區間和 + 單點改」了，而且更短更快。需要升級到 Segment Tree 只有兩個理由：

1. **運算不可減**——max、min、gcd、矩陣乘。Fenwick 的 `prefix` 靠減法拼區間，這些都做不了。
2. **要區間更新**——「把 `[l, r]` 整段設成 v」「整段加 v 後還要問區間 max」。Fenwick 靠差分只能勉強做到「區間加 + 區間和」，再複雜就沒辦法。

代價是程式長好幾倍、常數大 2–4 倍、記憶體 4n。這篇收遞迴版、迭代版、以及兩種 lazy（區間加、區間賦值），全部對拍暴力陣列驗過。

## 核心：任何區間都能拆成 O(log n) 個節點

```txt
                     [0,7]
             ┌─────────┴─────────┐
          [0,3]                 [4,7]
        ┌───┴───┐             ┌───┴───┐
     [0,1]     [2,3]       [4,5]     [6,7]
     ┌─┴─┐     ┌─┴─┐       ┌─┴─┐     ┌─┴─┐
   [0]  [1]  [2]  [3]    [4]  [5]  [6]  [7]

查詢 [2,6]：從根往下，每個節點三選一
   完全在查詢範圍外 → 回傳單位元素，不再往下
   完全被查詢範圍蓋住 → 直接用預存的 t[v]，不再往下   ← 關鍵
   部分重疊 → 遞迴兩個孩子，合併結果

[2,6] 最後停在 [2,3] + [4,5] + [6] 這三個節點
```

「部分重疊」的節點每層最多 2 個（左邊界一個、右邊界一個），所以總共只會展開 O(log n) 個節點。

> [!important] 節點編號用 `v`、`2v`、`2v+1`，陣列要開 `4n` 不是 `2n`
> 這棵樹不一定是滿二元樹。n = 5 時遞迴切分會產生最深 index 超過 `2n` 的節點，開 `2n` 會越界。`4n` 是安全上界（迭代版才能用 `2n`，因為它的編號方式不同）。

## 模板一：遞迴版 — 單點更新 + 區間查詢

最通用的骨架。把 `+` 換成 `max`／`min`／`gcd`、把 `0` 換成對應的單位元素（`LLONG_MIN`／`LLONG_MAX`／`0`）就能改用途。

```cpp
// build: O(n)   update / query: O(log n)   Space: O(4n)
struct SegTree {
  int n;
  vector<long long> t;

  explicit SegTree(const vector<long long>& a) : n(a.size()), t(4 * a.size()) {
    build(1, 0, n - 1, a);
  }

  void build(int v, int l, int r, const vector<long long>& a) {
    if (l == r) {
      t[v] = a[l];
      return;
    }
    int m = (l + r) / 2;
    build(2 * v, l, m, a);
    build(2 * v + 1, m + 1, r, a);
    t[v] = t[2 * v] + t[2 * v + 1];
  }

  void update(int v, int l, int r, int pos, long long val) {
    if (l == r) {
      t[v] = val;
      return;
    }
    int m = (l + r) / 2;
    if (pos <= m) {
      update(2 * v, l, m, pos, val);
    } else {
      update(2 * v + 1, m + 1, r, pos, val);
    }
    t[v] = t[2 * v] + t[2 * v + 1];  // 回溯時重算自己
  }

  long long query(int v, int l, int r, int ql, int qr) {
    if (qr < l || r < ql) {
      return 0;  // 完全在外 → 單位元素（max 要改 LLONG_MIN）
    }
    if (ql <= l && r <= qr) {
      return t[v];  // 完全被蓋住 → 直接用
    }
    int m = (l + r) / 2;
    return query(2 * v, l, m, ql, qr) + query(2 * v + 1, m + 1, r, ql, qr);
  }

  // 對外的乾淨介面
  void update(int pos, long long val) { update(1, 0, n - 1, pos, val); }
  long long query(int l, int r) { return query(1, 0, n - 1, l, r); }
};
```

## 模板二：迭代版 — 短一半，但只做得了單點更新

葉節點直接放在 `t[n + i]`，父節點是 `t[i] = t[2i] + t[2i+1]`。查詢時從兩端往上爬，遇到「自己是右孩子／左孩子」就把該段收進答案。**記憶體只要 `2n`**，常數也小得多。

```cpp
// build: O(n)   update / query: O(log n)   Space: O(2n)
struct SegTreeIter {
  int n;
  vector<long long> t;

  explicit SegTreeIter(const vector<long long>& a) : n(a.size()), t(2 * a.size()) {
    for (int i = 0; i < n; ++i) {
      t[n + i] = a[i];
    }
    for (int i = n - 1; i > 0; --i) {
      t[i] = t[2 * i] + t[2 * i + 1];
    }
  }

  void update(int pos, long long val) {
    for (t[pos += n] = val; pos > 1; pos >>= 1) {
      t[pos >> 1] = t[pos] + t[pos ^ 1];  // pos ^ 1 就是兄弟節點
    }
  }

  long long query(int l, int r) {  // 閉區間 [l, r]
    long long res = 0;
    for (l += n, r += n + 1; l < r; l >>= 1, r >>= 1) {
      if (l & 1) {
        res += t[l++];  // l 是右孩子，它的父節點會蓋到範圍外，先收下自己
      }
      if (r & 1) {
        res += t[--r];
      }
    }
    return res;
  }
};
```

> [!warning] 迭代版做不了 lazy，而且合併運算必須可交換
> `query` 是從左右兩端同時往中間收，收集順序不是由左到右——對 `+`／`max` 沒差，但對**矩陣乘法**這種不可交換的運算會算錯（要改成左右各存一個累積值最後再合）。需要 lazy 或不可交換運算，就回去用遞迴版。

## 模板三：Lazy — 區間加 + 區間和

區間更新的問題是「整段 `[l, r]` 加 v」可能碰到 10⁵ 個葉節點。Lazy 的想法是：**走到一個完全被蓋住的節點就停下來，在它身上留一張欠條，先不往下發**。

```txt
對 [0,3] 整段加 5，走到節點 [0,3] 發現完全被蓋住：
  該節點的 t  += 5 * 4    ← 自己的答案立刻算對（區間長度 4）
  該節點的 lz += 5        ← 欠條：我的子孫還沒收到這個 +5
  直接 return，不往下走

之後查詢 [0,1] 經過 [0,3] 時，先 push()：
  把 lz 發給兩個孩子（各自更新 t 和 lz），自己的 lz 歸零
```

```cpp
// update / query: O(log n)   Space: O(4n) × 2
struct LazyAdd {
  int n;
  vector<long long> t, lz;

  explicit LazyAdd(const vector<long long>& a)
      : n(a.size()), t(4 * a.size()), lz(4 * a.size(), 0) {
    build(1, 0, n - 1, a);
  }

  void build(int v, int l, int r, const vector<long long>& a) {
    if (l == r) {
      t[v] = a[l];
      return;
    }
    int m = (l + r) / 2;
    build(2 * v, l, m, a);
    build(2 * v + 1, m + 1, r, a);
    t[v] = t[2 * v] + t[2 * v + 1];
  }

  void applyAdd(int v, int l, int r, long long val) {
    t[v] += val * (r - l + 1);  // 區間和：要乘上長度
    lz[v] += val;               // 加法可疊加，所以是 +=
  }

  void push(int v, int l, int r) {
    if (lz[v] == 0) {
      return;
    }
    int m = (l + r) / 2;
    applyAdd(2 * v, l, m, lz[v]);
    applyAdd(2 * v + 1, m + 1, r, lz[v]);
    lz[v] = 0;
  }

  void update(int v, int l, int r, int ql, int qr, long long val) {
    if (qr < l || r < ql) {
      return;
    }
    if (ql <= l && r <= qr) {
      applyAdd(v, l, r, val);  // 蓋住了 → 留欠條就走
      return;
    }
    push(v, l, r);  // 要往下之前，先把欠條發掉
    int m = (l + r) / 2;
    update(2 * v, l, m, ql, qr, val);
    update(2 * v + 1, m + 1, r, ql, qr, val);
    t[v] = t[2 * v] + t[2 * v + 1];
  }

  long long query(int v, int l, int r, int ql, int qr) {
    if (qr < l || r < ql) {
      return 0;
    }
    if (ql <= l && r <= qr) {
      return t[v];
    }
    push(v, l, r);  // 查詢一樣要 push
    int m = (l + r) / 2;
    return query(2 * v, l, m, ql, qr) + query(2 * v + 1, m + 1, r, ql, qr);
  }

  void update(int l, int r, long long val) { update(1, 0, n - 1, l, r, val); }
  long long query(int l, int r) { return query(1, 0, n - 1, l, r); }
};
```

> [!important] `t[v]` 永遠是「已經套用了自己的 lz 之後」的正確值
> 這是整個 lazy 最容易搞混的地方。`lz[v]` 欠的是**子孫**，不是自己——`applyAdd` 裡 `t[v]` 當場就更新了。所以「完全被蓋住」時可以直接回傳 `t[v]` 而不用先 push，而「要往下走」時才必須 push。搞反成「lz 也欠自己」，查詢就會少算。

## 模板四：Lazy — 區間賦值 + 區間最大

換一種 lazy 就能看出模式：**標記怎麼疊加，取決於運算本身**。加法可疊加（`lz += val`），賦值則是後蓋前（`lz = val`），而且需要區分「有沒有標記」——這裡用 `optional` 表達，比用哨兵值安全。

```cpp
// update / query: O(log n)
struct LazyAssign {
  int n;
  vector<long long> t;
  vector<optional<long long>> lz;  // nullopt = 沒有待發的賦值

  explicit LazyAssign(const vector<long long>& a)
      : n(a.size()), t(4 * a.size()), lz(4 * a.size()) {
    build(1, 0, n - 1, a);
  }

  void build(int v, int l, int r, const vector<long long>& a) {
    if (l == r) {
      t[v] = a[l];
      return;
    }
    int m = (l + r) / 2;
    build(2 * v, l, m, a);
    build(2 * v + 1, m + 1, r, a);
    t[v] = max(t[2 * v], t[2 * v + 1]);
  }

  void applyAssign(int v, long long val) {
    t[v] = val;  // 區間最大：整段變成同一個值，最大值就是它，不用乘長度
    lz[v] = val;
  }

  void push(int v) {
    if (!lz[v]) {
      return;
    }
    applyAssign(2 * v, *lz[v]);
    applyAssign(2 * v + 1, *lz[v]);
    lz[v].reset();
  }

  void update(int v, int l, int r, int ql, int qr, long long val) {
    if (qr < l || r < ql) {
      return;
    }
    if (ql <= l && r <= qr) {
      applyAssign(v, val);
      return;
    }
    push(v);
    int m = (l + r) / 2;
    update(2 * v, l, m, ql, qr, val);
    update(2 * v + 1, m + 1, r, ql, qr, val);
    t[v] = max(t[2 * v], t[2 * v + 1]);
  }

  long long query(int v, int l, int r, int ql, int qr) {
    if (qr < l || r < ql) {
      return LLONG_MIN;  // max 的單位元素
    }
    if (ql <= l && r <= qr) {
      return t[v];
    }
    push(v);
    int m = (l + r) / 2;
    return max(query(2 * v, l, m, ql, qr), query(2 * v + 1, m + 1, r, ql, qr));
  }

  void update(int l, int r, long long val) { update(1, 0, n - 1, l, r, val); }
  long long query(int l, int r) { return query(1, 0, n - 1, l, r); }
};
```

> [!tip] 換運算時要同時改四個地方，漏一個就錯
> `build` 的合併、`update` 回溯的合併、`query` 的合併、以及「完全在外」回傳的**單位元素**。加法是 `0`、max 是 `LLONG_MIN`、min 是 `LLONG_MAX`、gcd 是 `0`。另外 `applyXxx` 裡要不要乘區間長度也跟著變——**和要乘，max／min 不用**。

## 常見陷阱

> [!warning] 六個會讓 Segment Tree 出錯的地方
> - **陣列開 `4n` 之外的大小**：遞迴版開 `2n` 或 `3n` 會在某些 n 越界，而且是 UB，可能跑對可能 segfault。
> - **`query` 忘記 push**：查詢也會往下走，不 push 就讀到過期的 `t`。這是最常見的 lazy bug。
> - **以為 `lz[v]` 也欠自己**：`t[v]` 已經是正確值，見模板三的 callout。
> - **區間加時 `applyAdd` 忘了乘區間長度**：單點更新的樹沒有這個問題，一改成 lazy 就冒出來。
> - **max 的單位元素寫 0**：全負數的陣列會回傳 0。跟 [[0124-Binary-Tree-Maximum-Path-Sum]] 的 `ans` 不能初始化成 0 是同一個坑。
> - **迭代版拿去做 lazy 或不可交換運算**：見模板二的 callout。

## 怎麼選

| 需求 | 用什麼 |
| --- | --- |
| 區間和 + 單點改 | [[Fenwick-Tree]]，短、快、記憶體省 |
| 區間和 + 區間加 | Fenwick 兩棵樹版也行；要再複雜就 LazyAdd |
| 區間 max／min／gcd + 單點改 | 模板二迭代版（最省） |
| 區間 max／min + 區間賦值／區間加 | 模板四 lazy |
| 靜態陣列、只查詢不更新的 max／min | [[Sparse-Table-RMQ]]，O(1) 查詢 |
| 不可交換運算（矩陣乘） | 模板一遞迴版 |

## Related Problems

- [[0307-Range-Sum-Query-Mutable]] — 單點改 + 區間和，Fenwick 或模板一都可
- [[0699-Falling-Squares]] — 區間賦值 + 區間 max，模板四原型
- [[2286-Booking-Concert-Tickets-in-Groups]] — 同時要 max 和 sum 的線段樹
- [[Fenwick-Tree]] — 大多數「區間和」題目的更好選擇
- [[Sparse-Table-RMQ]] — 靜態不更新時的 O(1) 查詢版本
