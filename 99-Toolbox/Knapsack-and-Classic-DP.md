---
tags:
  - dynamic-programming
  - pattern
dg-publish: true
---

## 這篇在解決什麼

背包是 DP 的骨幹題型，而它的三個變種（每件用一次／無限次／限量）在滾動陣列之後，**程式碼的差別只有迴圈方向和一層拆分**。這篇把那個差別講清楚——它是整個背包家族唯一需要想的地方——再收 LIS 的 O(n log n) 版本。

全部對拍驗過：0/1 背包對照 2ⁿ 暴力枚舉、完全背包對照 DP 定義式、有界背包對照展開後的暴力、LIS 兩種定義各對照 O(n²)，合計 3000 + 30000 組全過。

## 核心：滾動之後，迴圈方向決定了「能不能重複用」

二維的定義是 `dp[i][j]` =「只考慮前 i 件、容量 j」的最佳值。因為第 i 列只依賴第 i-1 列，可以壓成一維——但壓完之後，`dp[j - w]` 到底是「上一列的」還是「這一列的」，就取決於掃描方向：

```txt
dp[j] = max(dp[j], dp[j - w] + v)
                    └──┬──┘
                  這個值是誰？

j 由大到小（倒序）：j - w < j 還沒被這一輪碰過
                   ⇒ 讀到的是「上一列」＝ 還沒放過這件
                   ⇒ 每件最多用一次      → 0/1 背包

j 由小到大（正序）：j - w < j 已經被這一輪更新過
                   ⇒ 讀到的是「這一列」＝ 可能已經放了這件
                   ⇒ 同一件可以再放      → 完全背包
```

> [!important] 這兩行是整個背包家族的分水嶺
> 0/1 和完全背包的程式碼**只差 `for` 的方向**，其他一字不改。寫錯方向不會編譯失敗、不會 crash，只會得到「另一種背包」的答案——0/1 題目寫成正序會偷偷允許重複拿，答案偏大。記住這條，比記兩份模板有用。

## 0/1 背包 — 每件最多用一次

```cpp
// Time: O(nW)   Space: O(W)
long long knapsack01(int W, const vector<int>& w, const vector<long long>& v) {
  vector<long long> dp(W + 1, 0);
  for (size_t i = 0; i < w.size(); ++i) {
    for (int j = W; j >= w[i]; --j) {  // 倒序
      dp[j] = max(dp[j], dp[j - w[i]] + v[i]);
    }
  }
  return dp[W];
}
```

`j >= w[i]` 這個下界順便省掉了「裝不下」的判斷。

## 完全背包 — 每件無限次

```cpp
// Time: O(nW)   Space: O(W)
long long knapsackComplete(int W, const vector<int>& w, const vector<long long>& v) {
  vector<long long> dp(W + 1, 0);
  for (size_t i = 0; i < w.size(); ++i) {
    for (int j = w[i]; j <= W; ++j) {  // 正序
      dp[j] = max(dp[j], dp[j - w[i]] + v[i]);
    }
  }
  return dp[W];
}
```

## 有界背包 — 每件限量 c 個，用二進位拆分

樸素做法是「把第 i 件複製 c 份丟進 0/1 背包」，O(W·Σc)。二進位拆分把 c 拆成 `1, 2, 4, …, 剩餘`，只要 O(log c) 份就能組出 0..c 的任何數量：

```txt
c = 13  →  拆成 1, 2, 4, 6
           想拿 0 個：都不選
           想拿 5 個：1 + 4
           想拿 11 個：1 + 4 + 6
           0..13 每個數量都湊得出來，而且只用了 4 份而非 13 份
```

```cpp
// Time: O(W · Σ log c_i)   Space: O(W)
long long knapsackBounded(int W, const vector<int>& w, const vector<long long>& v,
                          const vector<int>& cnt) {
  vector<int> w2;
  vector<long long> v2;
  for (size_t i = 0; i < w.size(); ++i) {
    int c = cnt[i];
    for (int k = 1; c > 0; k <<= 1) {
      int take = min(k, c);  // 最後一份是剩餘量，不一定是 2 的冪
      w2.push_back(w[i] * take);
      v2.push_back(v[i] * take);
      c -= take;
    }
  }
  return knapsack01(W, w2, v2);  // 拆完就是普通的 0/1
}
```

> [!tip] 最後那份必須是「剩餘量」而不是下一個 2 的冪
> `c = 13` 拆成 `1, 2, 4, 8` 的話總和是 15 > 13，會允許拿超過上限。正確做法是 `min(k, c)` 讓最後一份補齊差額（例子裡的 6）。這個 off-by-one 在 `c` 剛好是 2ᵏ−1 時看不出來，其他值就會多拿。

## 計數版：把 `max` 換成 `+`

「有幾種方法湊出金額 W」是同一個骨架，只是合併運算變了。**但迴圈的內外層順序這時候有意義了**：

```cpp
// 組合數（不管順序，{1,2} 和 {2,1} 算同一種）：物品在外層
vector<long long> dp(W + 1, 0);
dp[0] = 1;
for (int coin : coins) {
  for (int j = coin; j <= W; ++j) {
    dp[j] += dp[j - coin];
  }
}

// 排列數（順序不同算不同）：容量在外層
vector<long long> dp(W + 1, 0);
dp[0] = 1;
for (int j = 1; j <= W; ++j) {
  for (int coin : coins) {
    if (coin <= j) {
      dp[j] += dp[j - coin];
    }
  }
}
```

> [!warning] 求最大／最小值時迴圈順序無所謂，求方案數時決定答案
> `max` 版兩種順序都對（因為 max 不在乎同一個集合被算幾次）。**計數版不是**：物品在外層代表「每種物品只被考慮一次」⇒ 組合；容量在外層代表「每一步都能選任何物品」⇒ 排列。CSES 的 *Coin Combinations I / II* 就是這一組對照題，兩題只差這個順序。

## LIS — O(n log n)

維護 `tail[k]` =「長度 k+1 的遞增子序列，結尾最小可能是多少」。這個陣列必然遞增，所以能二分。**`tail` 的長度就是答案，但 `tail` 的內容不是那個子序列**。

```cpp
// Time: O(n log n)   Space: O(n)
int lisStrict(const vector<int>& a) {  // 嚴格遞增
  vector<int> tail;
  for (int x : a) {
    auto it = lower_bound(tail.begin(), tail.end(), x);
    if (it == tail.end()) {
      tail.push_back(x);  // x 比所有結尾都大 → 接出更長的
    } else {
      *it = x;            // 把第一個 >= x 的結尾換小，讓後續更好接
    }
  }
  return tail.size();
}
```

> [!warning] `lower_bound` 和 `upper_bound` 選錯，75% 的測資會錯
> - **嚴格遞增**（`a < b`）→ `lower_bound`（找第一個 **≥ x** 的位置換掉）
> - **非嚴格遞增**（`a <= b`，允許相等）→ `upper_bound`（找第一個 **> x** 的位置）
>
> 實測 30000 組含大量重複值的隨機陣列：兩版各自對照 O(n²) 定義式都 0 錯，但**拿去比另一種定義時 22579 組不符**。最小例子 `a = {1,1,1,1}`：嚴格 LIS 是 1，非嚴格是 4。
>
> 「最長不下降子序列」「最少幾個遞減序列能覆蓋」這類題目讀清楚要哪一種，是這題唯一的難點。

## 常見陷阱

> [!warning] 六個地方
> - **0/1 寫成正序**：變成完全背包，答案偏大。
> - **有界背包拆分沒補剩餘量**：允許拿超過上限。
> - **計數題迴圈順序寫反**：組合數變排列數（或反過來）。
> - **`dp` 用 `int` 存方案數**：計數題常常要對 10⁹+7 取模，見 [[Modular-Arithmetic]]；不取模的話 `long long` 也可能爆。
> - **LIS 用錯 `lower_bound` / `upper_bound`**：見上面的 callout。
> - **初始值 `0` vs `-INF`**：「容量恰好裝滿」的題目要把 `dp[1..W]` 初始化成 `-INF`（表示不可達），只有 `dp[0] = 0`。初始化成全 0 代表「不必裝滿」，是不同的題目。

## 典型變形

- **恰好裝滿**：初始 `dp[0]=0`、其餘 `-INF`
- **最少物品數**：`max` 換 `min`，初始 `dp[0]=0`、其餘 `INF`
- **記錄方案**：多開一個 `from[]` 記轉移來源，最後回溯
- **二維費用**（重量 + 體積）：`dp[j][k]`，兩層都要照對應方向掃
- **分組背包**：每組最多選一件，組在最外層、容量倒序、組內物品最內層
- **LIS 變形**：俄羅斯套娃（先排序再 LIS）、最長不下降、最少遞減序列覆蓋（= LIS 長度，Dilworth 定理）

## Related Problems

- [[0416-Partition-Equal-Subset-Sum]] — 0/1 背包的布林版
- [[0322-Coin-Change]] — 完全背包求最少件數
- [[0518-Coin-Change-II]] — 完全背包求組合數，物品在外層
- [[0377-Combination-Sum-IV]] — 名字叫組合，其實求排列數，容量在外層
- [[0300-Longest-Increasing-Subsequence]] — LIS 本體
- [[0354-Russian-Doll-Envelopes]] — 排序 + LIS
- [[Modular-Arithmetic]] — 計數題的取模
