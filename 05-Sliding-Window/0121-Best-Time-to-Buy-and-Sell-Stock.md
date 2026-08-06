---
leetcode-id: 121
difficulty: easy
tags:
  - dynamic-programming
  - array
  - grind-169
  - neetcode-150
memo: 賣點固定時最佳買點必然是它左邊的最小值，所以掃一遍只要維護前綴最小值；把 min 的更新放在算獲利之前，就不需要哨兵值、也不可能發生減法溢位
dg-publish: true
---

## Problem Description

You are given an array `prices` where `prices[i]` is the price of a given stock on the `ith` day.

You want to maximize your profit by choosing a single day to buy one stock and choosing a different day in the future to sell that stock.

Return the maximum profit you can achieve from this transaction. If you cannot achieve any profit, return `0`.

## Solution

核心觀念：題目要「選兩天」，看起來是 O(n²) 的兩兩配對。但**只要把賣點固定下來，最佳買點就沒有選擇餘地——必然是它左邊的最小值**。於是「選兩天」塌縮成「掃一遍，每天問一句：如果今天賣，最多賺多少」，而回答這句話只需要一個變數。

```txt
prices        7   1   5   3   6   4
前綴最小值    7   1   1   1   1   1
今天賣獲利    0   0   4   2   5   3    ← 逐項取 max = 5
                              ↑
                        第 4 天賣、第 1 天買
```

> [!important] 化簡的關鍵是「固定一端」
> 面對「從序列中選 i < j 使某式最大」的題型，先別想怎麼同時挑兩個。**固定右端 j，問「此時最佳的 i 是什麼」**——如果那個答案只跟 `[0, j)` 的某個聚合量有關（最小值、最大值、前綴和⋯⋯），就能邊掃邊維護，O(n²) 直接降成 O(n)。
>
> 本題的聚合量是前綴最小值。[[0042-Trapping-Rain-Water]] 是前綴／後綴最大值，[[0053-Maximum-Subarray]] 是前綴和的最小值——同一個模子。

### 方法一：維護前綴最小值 — O(n)／O(1)（推薦）

```cpp
// Time: O(n)
// Space: O(1)
class Solution {
 public:
  int maxProfit(vector<int>& prices) {
    int ans = 0, mn = INT_MAX;
    for (int p : prices) {
      mn = min(mn, p);              // 先更新最小值
      ans = max(ans, p - mn);       // 再算「今天賣」的獲利
    }
    return ans;
  }
};
```

`mn` 初始化成 `INT_MAX` 就不需要「第一輪特判」——第一次 `min` 必定把它換成真實價格。

> [!warning] `min` 和 `max` 的順序決定會不會溢位
> 兩種順序都算得出正確答案，但安全性不同：
>
> ```cpp
> mn = min(mn, p);  ans = max(ans, p - mn);   // ✓ 更新後 mn <= p，p - mn >= 0，永不溢位
> ans = max(ans, p - mn);  mn = min(mn, p);   // ✗ 第一輪算 p - INT_MAX
> ```
>
> 第二種在本題**剛好**安全，因為 constraints 保證 `0 <= prices[i]`，`p - INT_MAX` 最小是 `INT_MIN + 1`。但只要價格可能為負（其他題、其他資料），`-2 - INT_MAX` 就是 signed overflow，也就是 UB。
>
> **把 `min` 提前，這個問題從根本上不存在**——不是靠 constraints 保佑，是靠不變量。順序改變也不影響正確性：`p` 自己是新最小值時 `p - mn = 0`，而 `ans` 本來就從 0 起跳。

> [!tip] 哨兵值 vs `INT_MAX`
> 另一種常見寫法是拿不可能出現的值（例如 `-1`）當哨兵，第一輪特判掉：
>
> ```cpp
> int prevmin = -1;
> for (int p : prices) {
>   if (prevmin == -1) { prevmin = p; continue; }   // 只在第一輪為真，卻要判斷 n 次
>   ...
> }
> ```
>
> 這在本題正確（價格保證 `>= 0`），但有三個缺點：**依賴「-1 不是合法輸入」這個外部前提**、**迴圈內多一個永遠為假的分支**、**意圖不明確**（讀者得回去查 constraints 才知道為什麼是 -1）。用 `INT_MAX` 當「還沒看到任何價格」的初始值，語意自洽，也不必特判。

### 方法二：滑動視窗雙指標 — O(n)／O(1)

`l` 是買點、`r` 是賣點。一旦發現更便宜的買點，左界**直接跳過去**而不是逐格移動：

```cpp
// Time: O(n)
// Space: O(1)
class Solution {
 public:
  int maxProfit(vector<int>& prices) {
    int ans = 0, n = prices.size();
    for (int l = 0, r = 1; r < n; ++r) {
      if (prices[r] < prices[l]) l = r;                  // 更便宜的買點
      else ans = max(ans, prices[r] - prices[l]);
    }
    return ans;
  }
};
```

這是本題被歸在 Sliding Window 的原因，也是「**窗口左界不一定一格一格移動**」的好範例——大多數滑動窗口題的 `l` 是 `while` 迴圈裡慢慢推進的，這題是直接跳。

> [!note] 它跟方法一是同一個演算法
> `prices[l]` 就是前綴最小值，`l = r` 就是 `mn = min(mn, p)`。換個說法而已，計算量完全相同。**知道它們等價比記住兩種寫法更重要**——否則會誤以為滑動窗口是另一套需要背的東西。

### 方法三：差分陣列上的 Kadane — O(n)／O(1)

第 `i` 天買、第 `j` 天賣的獲利，等於相鄰價差的連續和：

$$\text{prices}[j] - \text{prices}[i] = \sum_{k=i+1}^{j} \bigl(\text{prices}[k] - \text{prices}[k-1]\bigr)$$

所以**本題 ＝ 差分陣列的最大子陣列和**，也就是 [[0053-Maximum-Subarray]]：

```cpp
// Time: O(n)
// Space: O(1)
class Solution {
 public:
  int maxProfit(vector<int>& prices) {
    int ans = 0, cur = 0, n = prices.size();
    for (int i = 1; i < n; ++i) {
      cur = max(0, cur + prices[i] - prices[i - 1]);     // 累積為負就歸零重啟
      ans = max(ans, cur);
    }
    return ans;
  }
};
```

> [!tip] 三種寫法的對應關係
>
> ```txt
> 方法一  mn = min(mn, p)          更新前綴最小值
> 方法二  l = r                    左界跳到新買點
> 方法三  cur = max(0, ...)        累積獲利歸零重啟
> ```
>
> 三行講的是同一件事：**之前的買點已經不划算了，從這裡重新開始。**認出這點，這題就不再是三種解法，而是一個想法的三種語言。

三種寫法實測（30 萬組隨機測資含大量重複值與 0，加上 `n=1`、全遞減、全相等等邊界）全數通過。**方法一是首選**：最短、不需要額外概念、且溢位安全性是結構性的而非靠 constraints。

## Related Problems

[[0053-Maximum-Subarray]] — 差分之後兩題**完全等價**，Kadane 的「歸零重啟」就是本題的「換買點」
[[0122-Best-Time-to-Buy-and-Sell-Stock-II]] — 可交易無限次，退化成貪心地把所有正差分加總
[[0123-Best-Time-to-Buy-and-Sell-Stock-III]] — 限制兩次交易，這裡才需要真正的狀態機 DP
[[0042-Trapping-Rain-Water]] — 同樣是「固定右端、維護前綴極值」的形狀，只是維護的是最大值
[[0739-Daily-Temperatures]] — 同為「固定一端問左邊的某個聚合量」，但聚合量無法用單一變數表達，得改用單調堆疊
