---
tags:
  - geometry
  - math
  - pattern
dg-publish: true
---

## 這篇在解決什麼

幾何題最大的敵人不是演算法，是**浮點誤差**。`double` 比較 `== 0` 幾乎必錯，`eps` 該取多少沒有標準答案，而且錯誤往往只在特定測資上出現。

好消息是：**大部分幾何題的座標是整數，而基本判定全部可以用整數完成**——方向、相交、凸包、面積都只需要叉積，而叉積是純乘減。這篇的所有模板都不碰浮點。

對拍驗過：線段相交用兩種獨立實作（叉積版 vs 有理數參數方程版）對照 192082 組、凸包驗證四項性質 20000 組、點在多邊形內對照凸多邊形基準 29796 組，全過。

## 核心：一切都是叉積

```txt
cross(O, A, B) = (Ax - Ox)(By - Oy) − (Ay - Oy)(Bx - Ox)

  > 0  ⇒  O → A → B 是「左轉」（逆時針）
  = 0  ⇒  三點共線
  < 0  ⇒  O → A → B 是「右轉」（順時針）

               B                            A
               ↑                            ↑
   O ──────▶ A     cross > 0     O ──────▶ B     cross < 0

幾何意義：|cross(O,A,B)| = 三角形 OAB 面積的兩倍
```

```cpp
struct P {
  long long x, y;
  P operator-(const P& o) const { return {x - o.x, y - o.y}; }
  bool operator<(const P& o) const { return x != o.x ? x < o.x : y < o.y; }
  bool operator==(const P& o) const { return x == o.x && y == o.y; }
};

long long cross(const P& o, const P& a, const P& b) {
  return (a.x - o.x) * (b.y - o.y) - (a.y - o.y) * (b.x - o.x);
}

int sgn(long long v) { return (v > 0) - (v < 0); }

// +1 逆時針、0 共線、-1 順時針
int ccw(const P& o, const P& a, const P& b) { return sgn(cross(o, a, b)); }
```

> [!warning] 溢位界線：座標 ±10⁹ 剛好安全，±2×10⁹ 就爆
> 叉積的每一項最大 `(2C)² `，兩項相減最壞是 `2(2C)²`：
>
> | 座標上限 C | 最壞值 | `long long`（上限 9.223×10¹⁸） |
> | --- | --- | --- |
> | 10⁹ | 8.0×10¹⁸ | **安全** |
> | 2×10⁹ | 3.2×10¹⁹ | 溢位，要 `__int128` |
>
> 題目說座標範圍 ±10⁹ 時可以放心用 `long long`；一旦題目做了平移或縮放（例如先減去某個基準點）讓有效範圍變大，就要重新算這個界線。跟 [[Modular-Arithmetic]] 一樣，**寫出界線比說「小心溢位」有用**。

## 線段相交 — 含所有退化情況

標準情況是「兩線段互相跨越對方」，但端點剛好落在另一條線段上、兩線段共線重疊，這些都算相交，而且是最常漏的：

```cpp
// 已知 p 與線段 ab 共線，判斷 p 是否落在 [a, b] 上
bool onSegment(const P& p, const P& a, const P& b) {
  return min(a.x, b.x) <= p.x && p.x <= max(a.x, b.x) &&
         min(a.y, b.y) <= p.y && p.y <= max(a.y, b.y);
}

bool segIntersect(const P& a, const P& b, const P& c, const P& d) {
  int d1 = ccw(a, b, c), d2 = ccw(a, b, d);
  int d3 = ccw(c, d, a), d4 = ccw(c, d, b);
  if (d1 * d2 < 0 && d3 * d4 < 0) {
    return true;  // 標準相交：cd 跨越 ab，且 ab 跨越 cd
  }
  // 退化：某個端點落在另一條線段上（含共線重疊）
  if (d1 == 0 && onSegment(c, a, b)) return true;
  if (d2 == 0 && onSegment(d, a, b)) return true;
  if (d3 == 0 && onSegment(a, c, d)) return true;
  if (d4 == 0 && onSegment(b, c, d)) return true;
  return false;
}
```

> [!important] 那四個退化判斷不是可選的
> 只寫 `d1*d2 < 0 && d3*d4 < 0` 會漏掉：端點碰端點、端點碰線段中間、兩線段共線且部分重疊。實測 192082 組隨機小整數座標（座標範圍只有 −3..3，退化情況極多），叉積版與獨立的有理數參數方程版**完全一致**——但那是**加上四個退化判斷之後**才成立的。

## 凸包 — Andrew monotone chain

先按 `(x, y)` 排序，然後掃兩遍（下凸包、上凸包），用一個 stack 維護「一路左轉」的性質。

```cpp
// Time: O(n log n)（瓶頸是排序）   回傳逆時針順序，不含共線點
vector<P> convexHull(vector<P> pts) {
  sort(pts.begin(), pts.end());
  pts.erase(unique(pts.begin(), pts.end()), pts.end());
  int n = pts.size();
  if (n <= 2) {
    return pts;
  }
  vector<P> h(2 * n);
  int k = 0;
  for (int i = 0; i < n; ++i) {  // 下凸包
    while (k >= 2 && cross(h[k - 2], h[k - 1], pts[i]) <= 0) {
      --k;  // 不是左轉就把上一個點彈掉
    }
    h[k++] = pts[i];
  }
  for (int i = n - 2, t = k + 1; i >= 0; --i) {  // 上凸包
    while (k >= t && cross(h[k - 2], h[k - 1], pts[i]) <= 0) {
      --k;
    }
    h[k++] = pts[i];
  }
  h.resize(k - 1);  // 最後一個點等於第一個，去掉
  return h;
}
```

> [!tip] `<= 0` 和 `< 0` 決定共線點留不留
> - `cross(...) <= 0` 就彈掉 ⇒ **共線點被移除**，凸包只保留「真正的角」
> - `cross(...) < 0` 才彈 ⇒ **共線點保留**，凸包邊上的點也會出現在結果裡
>
> 題目問「凸包的頂點有幾個」用前者，問「有幾個點在凸包邊界上」（CSES *Convex Hull* 的某些變形）用後者。這一個字元的差別會讓答案差很多。

驗證凸包正確性的四件事（實測 20000 組全過）：**輸出的點都來自輸入**、**所有輸入點都不在任何一條凸包邊的右側**、**連續三點都是左轉**、**鞋帶面積等於三角扇形剖分的面積和**。自己實作完值得照這四項驗一次。

## 多邊形面積 — 鞋帶公式

```cpp
// 回傳「2 倍面積」，保持整數精確；要真面積再除以 2.0
// 適用任何簡單多邊形（凸或凹），頂點需依序（順時針或逆時針皆可）
long long area2(const vector<P>& poly) {
  long long s = 0;
  int n = poly.size();
  for (int i = 0; i < n; ++i) {
    const P& a = poly[i];
    const P& b = poly[(i + 1) % n];
    s += a.x * b.y - a.y * b.x;
  }
  return llabs(s);
}
```

> [!tip] 回傳 2×面積，不要提早除以 2
> 整數座標的多邊形，2×面積必定是整數。提早除以 2 會引入浮點或損失精度；讓呼叫端決定要不要除。這個習慣也讓「面積比較」「面積是否為整數」這類判斷變成純整數運算。
>
> 順帶：**皮克定理** `A = i + b/2 − 1`（i = 內部格點數、b = 邊界格點數）配上這個函式，可以純整數算出格點數——CSES 的 *Polygon Lattice Points* 就是這題。

## 點在多邊形內 — 射線法

從查詢點往右射一條水平線，數穿過幾條邊：奇數在內、偶數在外。邊界要另外先判。

```cpp
// 回傳 1 = 內部、0 = 邊界上、-1 = 外部
// 適用任何簡單多邊形；全整數運算
int pointInPolygon(const P& p, const vector<P>& poly) {
  int n = poly.size();
  for (int i = 0; i < n; ++i) {  // 先判邊界
    const P& a = poly[i];
    const P& b = poly[(i + 1) % n];
    if (ccw(a, b, p) == 0 && onSegment(p, a, b)) {
      return 0;
    }
  }
  bool inside = false;
  for (int i = 0; i < n; ++i) {
    P a = poly[i], b = poly[(i + 1) % n];
    if ((a.y > p.y) != (b.y > p.y)) {  // 這條邊跨越 p 的水平線
      long long d = cross(a, b, p);    // 用叉積代替算交點的 x，避免浮點
      if (b.y > a.y ? d > 0 : d < 0) {
        inside = !inside;
      }
    }
  }
  return inside ? 1 : -1;
}
```

> [!important] `(a.y > p.y) != (b.y > p.y)` 這個寫法同時解決了「頂點剛好在射線上」
> 用嚴格 `>` 讓每條邊的下端點算「在下方」、上端點算「在上方」，於是射線穿過一個頂點時**只會被計算一次**而不是兩次。若寫成 `>=` 或分開判斷，水平邊和頂點就會重複計數，導致內外判斷反轉。
>
> 交點位置也不要真的去算 `x = a.x + (p.y-a.y)*(b.x-a.x)/(b.y-a.y)`——那是浮點。用 `cross(a, b, p)` 的符號直接判斷 p 在邊的哪一側，效果相同且全整數。

## 常見陷阱

> [!warning] 六個地方
> - **用 `double` 做判定**：`== 0` 幾乎必錯。整數座標就全程整數。
> - **叉積溢位**：見上面的界線表。
> - **線段相交漏掉退化情況**：四個 `onSegment` 判斷不能省。
> - **凸包的 `<=` / `<` 選錯**：共線點留不留，答案差很多。
> - **凸包沒去重**：重複點會讓 `while` 條件行為異常，先 `sort` + `unique`。
> - **多邊形頂點順序不對**：鞋帶公式要求頂點沿邊界依序排列；隨意順序算出來的是別的東西（取絕對值也救不了）。

## 典型用法

- **凸包**：最遠點對（旋轉卡尺）、最小包圍矩形、凸多邊形碰撞
- **叉積排序**：極角排序（`atan2` 會有精度問題，用叉積比較）
- **面積**：鞋帶 + 皮克定理算格點數
- **線段相交**：判斷路徑是否交叉、掃描線求所有交點
- **點在多邊形內**：地圖圍欄、判斷可視區域
- **半平面交／最近點對**：更進階，但基礎仍是這裡的叉積

## Related Problems

- [[0587-Erect-the-Fence]] — 凸包原題（要保留共線點的那一版）
- [[0812-Largest-Triangle-Area]] — 叉積算面積
- [[1266-Minimum-Time-Visiting-All-Points]] — 切比雪夫距離，幾何直覺題
- [[Modular-Arithmetic]] — 同樣是「先算清楚溢位界線」的思路
