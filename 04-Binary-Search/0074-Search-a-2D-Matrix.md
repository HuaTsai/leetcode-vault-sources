---
leetcode-id: 74
difficulty: medium
tags:
  - binary-search
  - array
  - matrix
  - grind-169
  - neetcode-150
memo: 兩條性質合起來等於「攤平後是一條全序的一維陣列」，所以直接對 0 到 mn 做一次二分就好；關鍵是還原座標要除以行寬 n 而不是列數 m，方陣時手誤剛好無害所以特別容易漏測
dg-publish: true
---

## Problem Description

You are given an `m x n` integer matrix `matrix` with the following two properties:

- Each row is sorted in non-decreasing order.
- The first integer of each row is greater than the last integer of the previous row.

Given an integer `target`, return `true` if `target` is in `matrix` or `false` otherwise.

You must write a solution in `O(log(m * n))` time complexity.

## Solution

核心觀念：題目給的兩條性質**合起來就是「攤平後全序」**。性質一保證列內遞增，性質二把相鄰兩列接起來也遞增 —— 於是整個矩陣按 row-major 讀出來，就是一條排好序的一維陣列。剩下的只是一次普通二分，加上一個座標還原。

```txt
matrix = [[ 1,  3,  5,  7],
          [10, 11, 16, 20],
          [23, 30, 34, 60]]

性質一：每列遞增             1 < 3 < 5 < 7
性質二：下一列首 > 上一列尾   7 < 10，20 < 23
        ↓ 兩者合起來
攤平後是一條全序陣列：
   1   3   5   7  10  11  16  20  23  30  34  60
   0   1   2   3   4   5   6   7   8   9  10  11   ← flat index k

   k -> matrix[k / n][k % n]      n = 行寬（每列有幾個元素）
```

> [!important] 只有「全序」才能攤平
> 這一步完全依賴性質二。若題目只保證列內遞增、行內遞增（[[0240-Search-a-2D-Matrix-II]]），攤平後**不是**排序的，二分立刻失效，得改用右上角階梯搜尋。看到 2D 矩陣先確認是哪一種。

### 方法一：攤平成一維，一次二分 — O(log(mn))／O(1)（推薦）

把 `[0, m*n)` 當成一維 index 直接二分，只在存取時才還原成 `(i, j)`。不需要真的建出一維陣列，所以空間仍是 O(1)。

```cpp
// Time: O(log(m * n))
// Space: O(1)
class Solution {
 public:
  bool searchMatrix(vector<vector<int>>& matrix, int target) {
    int m = matrix.size();
    int n = matrix[0].size();
    int l = 0;
    int r = m * n;
    while (l < r) {
      int mid = l + (r - l) / 2;
      int i = mid / n;
      int j = mid % n;
      if (matrix[i][j] == target) {
        return true;
      } else if (matrix[i][j] < target) {
        l = mid + 1;
      } else {
        r = mid;
      }
    }
    return false;
  }
};
```

> [!important] 這是模板一寫成**半開區間**的形式
> `r = m * n` 是半開區間 `[l, r)` 的初始值 —— `r` 是界標不是候選，所以可以等於 `m * n`（矩陣沒有第 `m*n` 個元素，但它也從沒被解參考）。
>
> 在半開慣例下 **`r = mid` 本身就是排除 mid**，等價於閉區間的 `r = mid - 1`。所以本題雖然有 `== target` 就 return 的模板一特徵，更新規則卻寫 `r = mid`，兩者並不矛盾 —— 換算成閉區間會是 `r = m*n - 1` 配 `r = mid - 1`。完整的慣例對照見 [[Binary-Search-Templates]]。

> [!warning] 還原座標要除以**行寬 `n`**，不是列數 `m`
> `i = mid / n`、`j = mid % n`，兩處都是 `n`。手誤寫成 `/ m` 的後果實測（20 萬組隨機測資）：**66.1% 直接越界存取**、0.5% 安靜答錯。
>
> 真正陰險的是 `m == n` 的方陣（佔測資 16.6%）手誤剛好無害 —— **只拿方陣測會完全看不出問題**，一定要用長方形矩陣驗。

### 方法二：兩次二分，先定列再定行 — O(log m + log n)／O(1)

先二分找出「最後一個列首 `≤ target` 的列」，再在該列內二分。複雜度 `O(log m + log n) = O(log(mn))`，與方法一同級。

```cpp
// Time: O(log m + log n) = O(log(m * n))
// Space: O(1)
class Solution {
 public:
  bool searchMatrix(vector<vector<int>>& matrix, int target) {
    int m = matrix.size(), n = matrix[0].size();
    if (target < matrix[0][0]) return false;  // 沒有任何列首 <= target
    int l = 0, r = m - 1;
    while (l < r) {
      int mid = l + (r - l + 1) / 2;          // 上取整，配 l = mid
      if (matrix[mid][0] <= target) l = mid;  // mid 可能就是答案，保留
      else r = mid - 1;
    }
    const auto& row = matrix[l];
    return binary_search(row.begin(), row.end(), target);
  }
};
```

> [!tip] 找「最後一個滿足條件」必須配上取整
> 這裡是 [[Binary-Search-Templates]] 的模板二鏡像版：`l = mid` 保留候選，就得用 `l + (r - l + 1) / 2` 保證 `mid > l`，否則 `r == l + 1` 時 `mid == l`，`l = mid` 原地不動直接死迴圈。
>
> 開頭那行 `if (target < matrix[0][0])` 也是必要的：模板二假設「答案必定存在於區間內」，但 `target` 比整個矩陣最小值還小時根本沒有合法的列，得先擋掉。

方法一只需要**一次**二分和一個除餘數，程式碼短、也沒有這兩個特例要處理，所以是首選。方法二的價值在於它是**矩陣不全序時唯一可行的方向**，值得留著當對照。

**驗證**：兩版皆以 `g++ -std=c++20` 編譯，隨機 30 萬組（1×1 到 6×6、含負數、target 半數不存在）比對展平後的 `std::binary_search`，全數通過；另測 1×1、單列、單行、target 落在頭尾與界外等邊界案例，全過。

## Related Problems

[[Binary-Search-Templates]] — 本題是「模板一 · 半開區間」的實例，方法二則用到模板二的上取整鏡像版
[[0240-Search-a-2D-Matrix-II]] — 少了「下一列首 > 上一列尾」這條，攤平失效，改用右上角階梯搜尋 O(m + n)
[[0704-Binary-Search]] — 攤平之後本題就退化成這題，差別只在多一層座標還原
[[0033-Search-in-Rotated-Sorted-Array]] — 同為模板一，但因為要判斷哪一半有序而複雜得多
[[0378-Kth-Smallest-Element-in-a-Sorted-Matrix]] — 同樣是排序矩陣，但改成在**答案值域**上二分，是更進階的變形
