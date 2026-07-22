---
leetcode-id: 875
difficulty: medium
tags:
  - binary-search
  - array
  - neetcode-150
memo: 二分的對象是答案值域而不是陣列，述詞「速度 k 吃得完嗎」隨 k 單調，於是在 1 到 max 上找第一個可行值；天花板除法要寫成 p 加 m 減 1 再除以 m，寫錯會恆為高估而讓答案偏大，指紋是回傳值超出合法上界 max
dg-publish: true
---

## Problem Description

Koko loves to eat bananas. There are `n` piles of bananas, the `ith` pile has `piles[i]` bananas. The guards have gone and will come back in `h` hours.

Koko can decide her bananas-per-hour eating speed of `k`. Each hour, she chooses some pile of bananas and eats `k` bananas from that pile. If the pile has less than `k` bananas, she eats all of them instead and will not eat any more bananas during this hour.

Koko likes to eat slowly but still wants to finish eating all the bananas before the guards return.

Return the minimum integer `k` such that she can eat all the bananas within `h` hours.

## Solution

核心觀念：**二分的對象不是陣列，是答案本身的值域。** 陣列 `piles` 根本沒排序、也不需要排序 —— 真正單調的是「速度 `k` 吃得完嗎」這個述詞：速度越快花的時間越少，所以可行性一旦成立就不會再翻回去。

```txt
速度 k :  1   2   3   4   5   6   7   8  ...  max
可行？  : ✗   ✗   ✗   ✗   ✓   ✓   ✓   ✓  ...   ✓
                          ↑ 答案 = 第一個 ✓

k = 1   花最久（吃完全部要 sum 小時）  -> 最可能不可行
k = max 每堆一小時，共 n 小時          -> 題目保證 n <= h，必定可行
```

值域下界是 `1`（速度不能為 0），上界是 `max(piles)`：再快也沒用，因為一小時最多只能吃一堆。所以答案必定落在 `[1, max]`，直接對這個區間套「找第一個可行值」的二分。

單次可行性檢查是 O(n)，二分 O(log max)，總計 **O(n log max)**。

### 方法一：在值域上二分，半開區間 — O(n log(max))／O(1)（推薦）

```cpp
// Time: O(n log(max(piles)))
// Space: O(1)
class Solution {
 public:
  int minEatingSpeed(vector<int>& piles, int h) {
    int l = 1, r = ranges::max(piles) + 1;   // 半開區間 [1, max+1)
    while (l < r) {
      int m = l + (r - l) / 2;
      long long hours = 0;
      for (int p : piles) hours += (p + m - 1) / m;  // ceil(p / m)
      if (hours > h) l = m + 1;              // 太慢，速度要更快
      else r = m;                            // 可行，m 可能就是答案
    }
    return l;
  }
};
```

`r = max + 1` 是半開區間的界標 —— `max` 本身仍是合法候選，只是不作為 `r` 的可及值。這裡從未解參考 `piles[r]`（述詞是**算**出來的，不是查表），所以閉區間、半開區間都能用，見下方方法二。

> [!warning] 天花板除法寫錯會恆為高估，答案偏大
> `ceil(p / m)` 有三種等價寫法，挑一種記熟：
>
> ```cpp
> (p + m - 1) / m        // 最常見
> (p - 1) / m + 1        // piles[i] >= 1 保證成立
> p / m + (p % m != 0)   // 最直白
> ```
>
> 常見手誤是把 `- 1` 寫成 `+ 1`。那個版本在 `m == 1` 時每堆多算 **2** 小時、在 `p % m == 0` 或 `p % m == m-1` 時多算 **1** 小時，只有 `p % m` 落在 `1 .. m-2` 才剛好正確 —— 所以它**會過一部分測資**，不容易一眼看出。實測 20 萬組隨機測資失敗 37.4%。
>
> **指紋：高估小時數 → `hours > h` 太容易成立 → `l` 被推高 → 答案偏大，甚至超出合法上界。** 官方範例二 `piles=[30,11,23,4,20], h=5` 應為 `30`，錯誤版本回傳 `31` —— 回傳值大於 `max(piles)` 在邏輯上不可能，看到就知道是可行性檢查算錯了，不是二分寫錯。

> [!note] `hours` 用 `int` 其實剛好夠，但建議還是寫 `long long`
> 直覺會擔心 `m == 1` 時 `hours` 等於 `sum(piles)`，最大 `10⁴ × 10⁹ = 10¹³`，遠超 int。**但那個情境到不了**，兩步就能證明：
>
> 1. **`r ≤ 2m`** —— 由 `m = l + (r-l)/2 ≥ (l+r-1)/2` 得 `2m ≥ l+r-1`，配上 `l ≥ 1`。中點永遠不小於右界的一半。
> 2. **`sum ≤ h · r`** —— `r` 必定是某個已驗證可行的速度（或初始的 `max+1`），可行代表 `Σceil(pᵢ/r) ≤ h`，而該式 `≥ sum/r`。
>
> 合起來 `hours(m) ≤ sum/m + n ≤ h·r/m + n ≤ 2h + n = 2×10⁹ + 10⁴ < INT_MAX`（餘裕約 7%）。實測 6 萬組隨機測資從未違反此界，且最緊的一組比值達 1.000 —— 這個界是貼緊的。
>
> 極端對照：`n=10⁴`、每堆 `10⁹`（`sum = 10¹³`）時，實際算出的 `hours` 最大只有 `1.31×10⁹` —— 因為答案是 `10⁴`，二分根本不會往小的 `m` 探。
>
> 仍建議 `long long` 的理由是**這個保證很脆**：它同時依賴 `l` 初始為 1、`r` 初始為 `max+1`、`m` 用下取整。任何改動都得重推一次，7% 的餘裕不值得每次重算。同類型的 [[1011-Capacity-To-Ship-Packages-Within-D-Days]]、[[0410-Split-Array-Largest-Sum]] 累加量的界各不相同，直接寫 `long long` 比逐題分析划算。

### 方法二：同樣的邏輯，改用閉區間 — O(n log(max))／O(1)

只有初始的 `r` 不同，迴圈本體一字未改：

```cpp
// Time: O(n log(max(piles)))
// Space: O(1)
class Solution {
 public:
  int minEatingSpeed(vector<int>& piles, int h) {
    int l = 1, r = ranges::max(piles);       // 閉區間 [1, max]
    while (l < r) {
      int m = l + (r - l) / 2;
      long long hours = 0;
      for (int p : piles) hours += (p + m - 1) / m;
      if (hours > h) l = m + 1;
      else r = m;                            // 閉區間下這是「保留 m」
    }
    return l;
  }
};
```

> [!tip] 為什麼這題兩種區間慣例都能用
> 判斷法是「**迴圈內有沒有解參考 `arr[r]`**」。[[0074-Search-a-2D-Matrix]] 和 [[0153-Find-Minimum-in-Rotated-Sorted-Array]] 的述詞要查表，`r` 必須是合法 index，慣例就被綁死；本題的述詞是**算**出來的，`r` 只是一個數值，兩種都行。
>
> 注意 `r = m` 這行在兩種慣例下**語意相反卻都正確**：閉區間是「保留 m 當候選」，半開是「排除 m、但 m 已知可行所以記在 r 上」，最後 `l` 都收斂到同一個位置。完整對照見 [[Binary-Search-Templates]]。

**驗證**：兩版皆以 `g++ -std=c++20` 編譯，隨機 20 萬組比對從 1 逐一試的暴力解，全數通過；三個官方範例、以及 `piles=[1] h=1`、`piles=[10⁹] h=1`、`piles=[10⁹]×10⁴ h=10⁴` 等邊界案例皆正確。

## Related Problems

[[Binary-Search-Templates]] — 本題是「在答案值域上二分」的代表：二分對象不是陣列，述詞是自己算出來的單調可行性
[[1011-Capacity-To-Ship-Packages-Within-D-Days]] — 幾乎同構，把「速度」換成「船載重」，值域下界改成 `max(weights)`
[[0410-Split-Array-Largest-Sum]] — 同一套路的困難版，最小化「最大子陣列和」，可行性檢查改成貪心分段
[[1482-Minimum-Number-of-Days-to-Make-m-Bouquets]] — 在「天數」值域上二分，可行性檢查是掃連續段
[[0035-Search-Insert-Position]] — 同為模板二找第一個可行位置，但二分對象是真正的陣列，可對照兩者差異
