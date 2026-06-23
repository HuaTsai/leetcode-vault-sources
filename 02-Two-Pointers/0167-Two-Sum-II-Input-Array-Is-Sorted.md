---
leetcode-id: 167
difficulty: medium
tags:
  - two-pointers
  - array
  - binary-search
  - neetcode-150
memo: 陣列已排序，用相向雙指針靠和的大小單調地收縮區間，O(n)／O(1)；和太大就移右指針、太小就移左指針，正好對應「向左變小、向右變大」的單調性
dg-publish: true
---

## Problem Description

Given a 1-indexed array of integers `numbers` that is already **_sorted in non-decreasing order_**, find two numbers such that they add up to a specific `target` number. Let these two numbers be `numbers[index1]` and `numbers[index2]` where `1 <= index1 < index2 <= numbers.length`.

Return the indices of the two numbers `index1` and `index2`, each incremented by one, as an integer array `[index1, index2]` of length 2.

The tests are generated such that there is **exactly one solution**. You may not use the same element twice.

Your solution must use only constant extra space.

## Solution

核心觀念：陣列「已排序」是這題的全部關鍵。從兩端往中間夾，`i` 指最小、`j` 指最大，看 `nums[i] + nums[j]` 與 `target` 的大小關係：和太大只能讓 `j` 左移（值變小），和太小只能讓 `i` 右移（值變大）。每一步都安全地排除掉一個元素，因此一趟掃描 O(n) 就能找到答案，且只用兩個指針、O(1) 空間。

```txt
nums:  [ 2,  7, 11, 15 ]   target = 9
         i           j      2 + 15 = 17 > 9 → j--
         i       j          2 + 11 = 13 > 9 → j--
         i   j              2 +  7 =  9 ✔ → 回傳 {i+1, j+1} = {1, 2}
```

> [!important] 為什麼移動指針不會錯過答案
> 假設目前 `nums[i] + nums[j] > target`。因為陣列遞增，`nums[i]` 是 `[i, j]` 區間裡最小的數，它配上 `nums[j]` 都已經超標，那 `nums[j]` 配上區間內任何更大的數只會更超標——`nums[j]` 不可能是解的一員，可以放心 `--j` 永久排除。和太小時 `++i` 同理。每步排除一個候選，所以 O(n) 必然收斂。

### 方法一：相向雙指針 — O(n)／O(1)（推薦）

```cpp
// Time: O(n)
// Space: O(1)
class Solution {
 public:
  vector<int> twoSum(vector<int>& nums, int target) {
    int i = 0;
    int j = nums.size() - 1;
    while (i < j) {
      int sum = nums[i] + nums[j];
      if (sum == target) {
        return {i + 1, j + 1};
      } else if (sum > target) {
        --j;
      } else {
        ++i;
      }
    }
    return {};
  }
};
```

> [!tip] 一點小整理
> 原本寫法在三個分支裡各算了一次 `nums[i] + nums[j]`，抽成區域變數 `sum` 算一次即可，省掉重複計算也更好讀。題目限制 `-1000 <= nums[i] <= 1000`，兩數和最多 2000，不會 `int` overflow。`i < j` 也天然保證了「不重複使用同一個元素」。

### 方法二：固定一數 + 二分搜尋 — O(n log n)／O(1)

沒想到雙指針時的退路：陣列既然有序，對每個 `nums[i]` 在它右邊的區間二分搜尋 `target - nums[i]`。比雙指針多一個 log，但仍是常數空間，也是「利用有序」的另一種直覺。

```cpp
// Time: O(n log n)
// Space: O(1)
class Solution {
 public:
  vector<int> twoSum(vector<int>& nums, int target) {
    int n = nums.size();
    for (int i = 0; i < n; ++i) {
      int need = target - nums[i];
      int lo = i + 1, hi = n - 1;  // 只在 i 右邊找，避免重複用到自己
      while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == need) return {i + 1, mid + 1};
        else if (nums[mid] < need) lo = mid + 1;
        else hi = mid - 1;
      }
    }
    return {};
  }
};
```

## Related Problems

[[0001-Two-Sum]] — 未排序版本，沒有單調性可利用，改用 hash map 以 O(n)／O(n) 換取查找
[[0125-Valid-Palindrome]] — 同為相向雙指針，從兩端往中間夾
[[0011-Container-With-Most-Water]] — 相向雙指針，靠「移動較矮的一邊」單調地縮短區間
[[0015-3Sum]] — 排序後固定一個數，對剩下區間套用本題的雙指針
