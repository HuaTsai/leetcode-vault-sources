---
leetcode-id: 746
difficulty: easy
tags:
  - dynamic-programming
  - neetcode-150
memo: f(i) 定義成站上第 i 階、尚未付 cost[i] 的最小花費，f(0) = f(1) = 0 對應可從 0 或 1 起步；樓頂是 index n，迴圈要跑到 i <= n
dg-publish: true
---

## Problem Description

You are given an integer array `cost` where `cost[i]` is the cost of `ith` step on a staircase.

Once you pay the cost, you can either climb one or two steps.

You can either start from the step with index 0, or the step with index 1.

Return the minimum cost to reach the top of the staircase, which is the position just past the last step (index `cost.length`).

## Solution

錢是在**離開**某一階時付的：付了 `cost[i]` 才能從第 i 階往上跨 1 或 2 階。所以把 `f(i)` 定義成「站上第 i 階、還沒付 `cost[i]`」的最小花費，站在第 i 階回頭看，最後一步只有兩種可能：從第 i−1 階付 `cost[i-1]` 跨 1 階上來，或從第 i−2 階付 `cost[i-2]` 跨 2 階上來，取便宜的那個：`f(i) = min(f(i-1) + cost[i-1], f(i-2) + cost[i-2])`。

可以從第 0 或第 1 階起步，代表站上這兩階都不用錢：`f(0) = f(1) = 0`。樓頂是最後一階再往上一格，也就是 index n，答案是 `f(n)`。

```txt
cost = [10, 15, 20]

i       :  0   1   2   3
cost[i] : 10  15  20   -     ← i = 3 是樓頂，沒有 cost
f(i)    :  0   0  10  15

f(2) = min(f(1) + 15, f(0) + 10) = min(15, 10) = 10
f(3) = min(f(2) + 20, f(1) + 15) = min(30, 15) = 15   ← 從第 1 階付 15 直接跨 2 階到頂
```

> [!important] 每一格只依賴前兩格
> 跟 [[0070-Climbing-Stairs]] 一樣不需要整個 dp 陣列，留兩個變數往前滾就好，空間從 O(n) 降到 O(1)。差別只在轉移從「兩個來源相加」換成「兩個來源各加上離開的成本後取 min」。

### 方法一：滾動變數 — O(n)／O(1)

`prev`、`cur` 分別是 `f(i-2)`、`f(i-1)`，起點都是 0。迴圈從 `i = 2` 跑到 `i = n`，結束時 `cur` 就是 `f(n)`。

```cpp
// Time: O(n)
// Space: O(1)
class Solution {
 public:
  int minCostClimbingStairs(vector<int>& cost) {
    int prev = 0;
    int cur = 0;
    for (int i = 2; i <= ssize(cost); ++i) {
      prev = exchange(cur, min(prev + cost[i - 2], cur + cost[i - 1]));
    }
    return cur;
  }
};
```

> [!warning] 樓頂是 index n，不是最後一階
> 迴圈條件是 `i <= ssize(cost)`。寫成 `i < ssize(cost)` 會停在 `f(n-1)`，也就是「站上最後一階」的花費，少算了離開最後一階（或倒數第二階）到樓頂的那一步。上面的例子會回傳 `f(2) = 10` 而不是 15。

> [!note] n = 2 不必特判
> 原本的寫法在開頭特判 `cost.size() == 2` 直接回傳 `min(cost[0], cost[1])`，也正確。但 n = 2 時迴圈剛好跑一圈（`i = 2`），算的就是 `min(0 + cost[0], 0 + cost[1])`，跟特判回傳的一樣，所以可以拿掉。起點定義對了，邊界就自然落在迴圈裡。

> [!tip] 另一種狀態定義：已經付了 `cost[i]`
> 把 `f(i)` 改成「站上第 i 階**並付了** `cost[i]`」的最小花費，轉移變成 `f(i) = cost[i] + min(f(i-1), f(i-2))`：
>
> ```cpp
> // Time: O(n)
> // Space: O(1)
> class Solution {
>  public:
>   int minCostClimbingStairs(vector<int>& cost) {
>     int prev = 0;
>     int cur = 0;
>     for (int c : cost) {
>       prev = exchange(cur, c + min(prev, cur));
>     }
>     return min(prev, cur);
>   }
> };
> ```
>
> 好處是可以用 range-for，沒有 `i - 2`、`i - 1` 的下標。代價是結尾要多一個 `min(prev, cur)`：樓頂可以從最後一階或倒數第二階跨上去，兩者都已經付過錢，取便宜的。漏掉這個 `min`、直接回傳 `cur` 是這個寫法最常見的錯。

## Related Problems

- [[0070-Climbing-Stairs]] — 同樣從前一、二階轉移，把「相加計數」換成「取 min 加成本」
- [[0198-House-Robber]] — 同為只看前兩格的 1-D DP，轉移改成「偷／不偷」取 max
- [[0064-Minimum-Path-Sum]] — 最小成本路徑的 2-D 版，來源從前兩階變成上方與左方
- [[0322-Coin-Change]] — 同樣是取 min 的轉移，可跨的步數從固定 1、2 變成任意面額
