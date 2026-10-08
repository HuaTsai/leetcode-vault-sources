---
leetcode-id: 198
difficulty: medium
tags:
  - dynamic-programming
  - grind-169
  - neetcode-150
memo: 搶或不搶第 i 間取 max；f(i) 定義成「只看前 i 間、第 i 間可搶可不搶」才只需要兩個變數，初始值設 0 就不必特判
dg-publish: true
---

## Problem Description

You are a professional robber planning to rob houses along a street. Each house has a certain amount of money stashed, the only constraint stopping you from robbing each of them is that adjacent houses have security systems connected and it will automatically contact the police if two adjacent houses were broken into on the same night.

Given an integer array `nums` representing the amount of money of each house, return the maximum amount of money you can rob tonight without alerting the police.

## Solution

相鄰兩間不能同時搶，所以站在第 i 間只有兩個選擇：**不搶**，答案就跟只看到第 i−1 間時一樣；**搶**，第 i−1 間就不能碰，拿 `nums[i]` 加上只看到第 i−2 間時的最大金額。兩者取大的。

把 `f(i)` 定義成「只看 `nums[0..i]` 這幾間時能搶到的最大金額，第 i 間**可搶可不搶**」，轉移就是 `f(i) = max(f(i-1), f(i-2) + nums[i])`，答案是 `f(n-1)`。

```txt
nums = [5, 1, 1, 5, 1]

i       :  0   1   2   3   4
nums[i] :  5   1   1   5   1
f(i)    :  5   5   6  10  10

f(1) = max(f(0), 0 + 1)    = max(5, 1)  = 5    ← 不搶第 1 間比較好
f(3) = max(f(2), f(1) + 5) = max(6, 10) = 10   ← 搶第 0、3 間
f(4) = max(f(3), f(2) + 1) = max(10, 7) = 10   ← 不搶最後一間
```

> [!important] `f(i)` 不要求搶第 i 間，所以只需要回頭看兩格
> 因為第 i 間可以不搶，`f(i-2)` 本身就已經是「第 i−2 間以前」的最大金額，不必再往前比。每一格只依賴前兩格，留兩個變數往前滾就好，空間 O(1)。狀態若改成「一定搶第 i 間」就得多看一格，見方法二。

### 方法一：第 i 間可搶可不搶 — O(n)／O(1)

`prev`、`cur` 分別是 `f(i-2)`、`f(i-1)`，起點都是 0，代表「還沒有任何房子」。結束時 `cur` 就是 `f(n-1)`。

```cpp
// Time: O(n)
// Space: O(1)
class Solution {
 public:
  int rob(vector<int>& nums) {
    int prev = 0;
    int cur = 0;
    for (int x : nums) {
      prev = exchange(cur, max(cur, prev + x));
    }
    return cur;
  }
};
```

> [!note] n = 1、n = 2 不必特判
> 起點是「還沒有房子」的 0 而不是 `nums[0]`，所以第一圈算的是 `max(0, 0 + nums[0])`，第二圈是 `max(nums[0], 0 + nums[1])`，邊界都落在迴圈裡。

> [!warning] 寫成陣列版時 `dp[1]` 是 `max(nums[0], nums[1])`，不是 `nums[1]`
> `dp[1]` 的意思是「只看前兩間的最大金額」，可以選第 0 間。寫成 `dp[1] = nums[1]` 等於強迫搶第 1 間，`[2, 1, 1, 2]` 會算出 3 而不是 4。

### 方法二：一定搶第 i 間 — O(n)／O(1)

把狀態換成 `g(i)` =「**一定搶**第 i 間」的最大金額。搶了第 i 間，上一間搶的只可能是第 i−2 或第 i−3 間：如果是第 i−4 間或更前面，中間的第 i−2 間可以補搶，金額非負所以不會更差。轉移是 `g(i) = max(g(i-2), g(i-3)) + nums[i]`。

最後一間不一定要搶，所以答案是 `max(g(n-1), g(n-2))`。

```txt
i       :  0   1   2   3   4
nums[i] :  5   1   1   5   1
f(i)    :  5   5   6  10  10    ← 可搶可不搶，不會變小，答案在最後一格
g(i)    :  5   1   6  10   7    ← 一定搶，g(1)、g(4) 被迫變小，答案要取最後兩格的 max
```

`pprev`、`prev`、`cur` 分別是 `g(i-3)`、`g(i-2)`、`g(i-1)`，起點同樣都是 0。

```cpp
// Time: O(n)
// Space: O(1)
class Solution {
 public:
  int rob(vector<int>& nums) {
    int pprev = 0;
    int prev = 0;
    int cur = 0;
    for (int x : nums) {
      int temp = max(pprev, prev) + x;
      pprev = prev;
      prev = cur;
      cur = temp;
    }
    return max(prev, cur);
  }
};
```

> [!warning] 兩種定義的初始值不要混用
> 原本的寫法特判 n ≤ 3，再把初始值設成 `pprev = nums[0]`、`prev = max(nums[0], nums[1])`、`cur = max(nums[0] + nums[2], nums[1])`，迴圈裡用的卻是 `g` 的轉移。後兩個初始值其實是 `f(1)`、`f(2)`，不是 `g(1) = nums[1]`、`g(2) = nums[0] + nums[2]`。
>
> 結果仍然正確，因為 `f(i-2) + nums[i]` 一定是不相鄰的合法方案。但這也說明手上既然已經是 `f`，直接用方法一的轉移就好，第三個變數和結尾的 `max` 都是多的。

## Related Problems

- [[0213-House-Robber-II]] — 房子圍成一圈，拆成「不含第一間」與「不含最後一間」兩段各跑一次本題
- [[0337-House-Robber-III]] — 搬到二元樹上，每個節點回傳「搶／不搶」兩個狀態
- [[0740-Delete-and-Earn]] — 按數值分桶加總後就是本題，相鄰數值不能同時拿
- [[0746-Min-Cost-Climbing-Stairs]] — 同為只看前兩格的 1-D DP，轉移是取 min 加成本
- [[0070-Climbing-Stairs]] — 同樣的滾動變數骨架，轉移是兩個來源相加
