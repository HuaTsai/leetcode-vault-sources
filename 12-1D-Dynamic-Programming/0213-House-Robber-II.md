---
leetcode-id: 213
difficulty: medium
tags:
  - dynamic-programming
  - neetcode-150
memo: 環拆成「不含最後一間」與「不含第一間」兩段各跑一次 0198 取 max；n＝1 要特判，否則兩段都是空的會回 0
dg-publish: true
---

## Problem Description

You are a professional robber planning to rob houses along a street. Each house has a certain amount of money stashed. All houses at this place are arranged in a circle. That means the first house is the neighbor of the last one. Meanwhile, adjacent houses have a security system connected, and it will automatically contact the police if two adjacent houses were broken into on the same night.

Given an integer array `nums` representing the amount of money of each house, return the maximum amount of money you can rob tonight without alerting the police.

## Solution

房子圍成一圈，多出來的限制只有一條：第 0 間和第 n−1 間也算相鄰，不能同時搶。所以任何合法方案**至少放掉頭尾其中一間**：放掉最後一間，剩下的 `nums[0..n-2]` 是一條直線；放掉第 0 間，剩下的 `nums[1..n-1]` 也是一條直線。兩段各跑一次 [[0198-House-Robber]]，取大的就是答案。

```txt
nums = [5, 1, 1, 5]

直線版（0198）  [5, 1, 1, 5]  → 搶第 0、3 間 = 10   ← 環上這兩間相鄰，不合法
不含最後一間    [5, 1, 1]     → 搶第 0、2 間 = 6
不含第 0 間        [1, 1, 5]  → 搶第 1、3 間 = 6

答案 = max(6, 6) = 6
```

> [!important] 拆成兩段不會漏、也不會多
> 不會漏：合法方案至少沒搶頭尾其中一間，把那一間拿掉後，它就是對應那一段的合法直線方案。不會多：每一段都少了頭或尾其中一間，段內不相鄰的方案放回環上也不會相鄰。「頭尾都不搶」的方案兩段都涵蓋到，重複算沒關係，反正是取 max。

### 方法一：拆成兩段各跑一次 House Robber — O(n)／O(1)

`robLine` 就是 0198 方法一原封不動，參數換成 `span`，讓呼叫端決定範圍。

```cpp
// Time: O(n)
// Space: O(1)
class Solution {
 public:
  int rob(vector<int>& nums) {
    if (nums.size() == 1) {
      return nums[0];
    }
    span<const int> s(nums);
    return max(robLine(s.first(s.size() - 1)), robLine(s.subspan(1)));
  }

 private:
  int robLine(span<const int> nums) {
    int prev = 0;
    int cur = 0;
    for (int x : nums) {
      prev = exchange(cur, max(cur, prev + x));
    }
    return cur;
  }
};
```

> [!warning] n = 1 要特判
> 只有一間時兩段都是空區間，各回 0，但那一間沒有鄰居，答案應該是 `nums[0]`。n = 2 不必特判：兩段分別是 `[nums[0]]` 和 `[nums[1]]`，取 max 剛好是答案。

> [!note] `span<const int>` 只是（指標，長度）的 view
> `first(n - 1)` 去掉最後一間、`subspan(1)` 去掉第 0 間，都不複製資料，所以空間還是 O(1)。不用 `span` 的話，改成 `robLine(nums, l, r)` 傳下標範圍也一樣。

### 方法二：單趟同時滾兩組狀態 — O(n)／O(1)

兩段只差頭尾各一間，所以可以在同一趟迴圈裡同時滾兩組 `{prev, cur}`：`rob0` 的範圍含第 0 間（對應 `[0, n-2]`），`norob0` 的範圍不含第 0 間（對應 `[1, n-1]`），用 `i != 0` 讓 `norob0` 跳過第 0 間。名字指的是「範圍有沒有包含第 0 間」，不是「一定搶第 0 間」。

`rob0` 沒有另外寫條件跳過最後一間，而是照樣吃進去，結束後退一格取 `rob0[0]`（prev），那正是只看到第 n−2 間的答案。

```cpp
// Time: O(n)
// Space: O(1)
class Solution {
 public:
  int rob(vector<int>& nums) {
    if (nums.size() == 1) {
      return nums[0];
    }
    array<int, 2> rob0{0, 0};  // prev, cur
    array<int, 2> norob0{0, 0};
    for (int i = 0; i < ssize(nums); ++i) {
      rob0[0] = exchange(rob0[1], max(rob0[0] + nums[i], rob0[1]));
      if (i != 0) {
        norob0[0] = exchange(norob0[1], max(norob0[0] + nums[i], norob0[1]));
      }
    }
    return max(rob0[0], norob0[1]);
  }
};
```

```txt
nums = [5, 1, 1, 5]          {prev, cur}

i  nums[i]   rob0       norob0
0     5      {0,  5}    {0, 0}    ← norob0 跳過第 0 間
1     1      {5,  5}    {0, 1}
2     1      {5,  6}    {1, 1}
3     5      {6, 10}    {1, 6}

rob0[0]   = 6    ← [0, n-2] 的答案
rob0[1]   = 10   ← 整條直線的答案，環上不合法
norob0[1] = 6    ← [1, n-1] 的答案
```

> [!warning] 回傳的是 `rob0[0]`，不是 `rob0[1]`
> 兩個都取 `[1]` 看起來比較對稱，但 `rob0[1]` 已經把最後一間算進去，等於沒有環的限制，上例會回 10。這個不對稱是單趟寫法的代價：省下一次迴圈的常數，複雜度不變，正確性卻藏在「多滾一格再退回來」裡。方法一兩邊都取 `cur`，比較不容易寫錯。

## Related Problems

- [[0198-House-Robber]] — 本題的直線版，方法一的 `robLine` 就是它
- [[0337-House-Robber-III]] — 搬到二元樹上，每個節點回傳「搶／不搶」兩個狀態
- [[0740-Delete-and-Earn]] — 按數值分桶加總後是直線版 House Robber
- [[0918-Maximum-Sum-Circular-Subarray]] — 同樣把環上的問題拆成幾種直線情況分別求解
