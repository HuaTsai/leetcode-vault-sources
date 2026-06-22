---
leetcode-id: 238
difficulty: medium
tags:
  - array
  - prefix-sum
  - grind-169
  - neetcode-150
memo: 用 prefix／suffix 乘積兩次掃描，關鍵是把 prefix[0] 與 suffix[n-1] 設為哨兵 1；可進一步用輸出陣列把額外空間壓到 O(1)
dg-publish: true
---

## Problem Description

Given an integer array `nums`, return an array `answer` such that `answer[i]` is equal to the product of all the elements of `nums` except `nums[i]`.

The product of any prefix or suffix of `nums` is guaranteed to fit in a 32-bit integer.

You must write an algorithm that runs in `O(n)` time and without using the division operation.

Follow up: Can you solve the problem in `O(1)` extra space complexity? (The output array does not count as extra space for space complexity analysis.)

## Solution

核心觀念：`answer[i]` = 「`i` 左邊所有元素的乘積」×「`i` 右邊所有元素的乘積」，也就是 prefix product 乘上 suffix product。只要 O(n) 預處理好兩邊乘積，就能 O(n) 組出答案，且全程不需要除法。

```txt
        idx:    0      1      2     ...    n-1
prefix[i] = product of nums[0 .. i-1]     (prefix[0]   = 1)
suffix[i] = product of nums[i+1 .. n-1]   (suffix[n-1] = 1)
ans[i]    = prefix[i] * suffix[i]
```

題目保證「任何 prefix 或 suffix 的乘積都落在 32-bit 內」，而整個陣列本身也是一個 prefix／suffix，所以總積同樣不會 overflow，陣列直接開 size `n` 即可。用哨兵值 `1` 起始，讓 `prefix[i]`、`suffix[i]` 的索引天然對齊到答案位置，最後一輪相乘就不需要任何特例判斷。

> [!important] 哨兵初始化是這個技巧的關鍵
> `prefix[0] = 1`、`suffix[n - 1] = 1`：最左邊的元素左邊「沒有東西」、最右邊的元素右邊「沒有東西」，乘法的單位元素是 `1`（不是 `0`，設 `0` 會把整個答案歸零）。有了這個哨兵，邊界 `ans[0]`、`ans[n - 1]` 就跟一般情況用同一條式子處理。方法二裡 `vector<int> ans(n, 1)`、`int prefix = 1`、`int suffix = 1` 也是同樣道理。

### 方法一：Prefix / Suffix 兩陣列 — O(n) space

```cpp
// Time: O(n)
// Space: O(n)
class Solution {
 public:
  vector<int> productExceptSelf(vector<int>& nums) {
    int n = nums.size();
    vector<int> prefix(n), suffix(n);
    prefix[0] = 1;
    for (int i = 1; i < n; ++i) prefix[i] = prefix[i - 1] * nums[i - 1];
    suffix[n - 1] = 1;
    for (int i = n - 2; i >= 0; --i) suffix[i] = suffix[i + 1] * nums[i + 1];
    vector<int> ans(n);
    for (int i = 0; i < n; ++i) ans[i] = prefix[i] * suffix[i];
    return ans;
  }
};
```

相較最初的寫法：陣列大小由 `n-1` 改回 `n`、並以哨兵值 `1` 起始，使 `prefix[i]`／`suffix[i]` 的索引天然對齊到答案位置，因此最後組合答案的迴圈不再需要 `i == 0`、`i == n-1` 的特例分支。

### 方法二：用輸出陣列滾動 — O(1) extra space（Follow up）

輸出陣列初始化為全 `1`，接著用 `prefix`、`suffix` 兩個滾動變數各掃一遍乘進去，額外空間降到 O(1)。兩個迴圈是對稱的同一個 pattern——先把累積值乘進 `ans[i]`，再用 `nums[i]` 更新累積值，只差掃描方向。

```cpp
// Time: O(n)
// Space: O(1) (output array 不計入)
class Solution {
 public:
  vector<int> productExceptSelf(vector<int>& nums) {
    int n = nums.size();
    vector<int> ans(n, 1);
    // 由左往右滾動 prefix，邊乘進 ans
    int prefix = 1;
    for (int i = 0; i < n; ++i) {
      ans[i] *= prefix;     // 累積到 i 左邊的乘積
      prefix *= nums[i];    // 更新成「i 及左邊」的乘積
    }
    // 由右往左滾動 suffix，邊乘進 ans
    int suffix = 1;
    for (int i = n - 1; i >= 0; --i) {
      ans[i] *= suffix;     // 再乘上 i 右邊的乘積
      suffix *= nums[i];    // 更新成「i 及右邊」的乘積
    }
    return ans;
  }
};
```

## Related Problems

[[0042-Trapping-Rain-Water]] — 同樣是預處理 prefix-max／suffix-max 兩個方向，再合併；也能進一步優化到 O(1) 額外空間
[[0724-Find-Pivot-Index]] — 比較左側與右側的 prefix／suffix 和找平衡點
[[0152-Maximum-Product-Subarray]] — 一樣處理連續乘積，需同時追蹤兩個方向的狀態
