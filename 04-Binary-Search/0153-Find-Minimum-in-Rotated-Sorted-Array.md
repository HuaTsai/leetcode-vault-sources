---
leetcode-id: 153
difficulty: medium
tags:
  - binary-search
  - array
  - grind-169
  - neetcode-150
memo: 關鍵是拿中點跟右端點比而不是左端點，因為右端點恆落在右段、語意才穩定，且下取整保證中點不會等於右端點而自比；本題屬「找邊界」型二分，中點本身可能就是答案，所以只能寫 r ＝ m，寫成 r ＝ m − 1 會安靜地答錯
dg-publish: true
---

## Problem Description

Suppose an array of length n sorted in ascending order is rotated between `1` and `n` times. For example, the array `nums = [0,1,2,4,5,6,7]` might become:

- `[4,5,6,7,0,1,2]` if it was rotated `4` times.
- `[0,1,2,4,5,6,7]` if it was rotated `7` times.

Notice that rotating an array `[a[0], a[1], a[2], ..., a[n-1]]` 1 time results in the array `[a[n-1], a[0], a[1], a[2], ..., a[n-2]]`.

Given the sorted rotated array `nums` of unique elements, return the minimum element of this array.

You must write an algorithm that runs in `O(log n) time`.

Constraints:

- All the integers of `nums` are unique.
- `nums` is sorted and rotated between `1` and `n` times.

## Solution

核心觀念：旋轉後的陣列一定是**兩段各自遞增，而且左段每個元素都大於右段每個元素**。答案就是右段的第一個，也就是那個唯一的斷點。二分的每一步只需要判斷「中點落在哪一段」。

```txt
   4  5  6  7 │ 0  1  2
   └─左段(大)─┘ └右段(小)┘
                ↑ 答案 = 右段第一個 = 唯一的斷點

   左段任一元素 > 右段任一元素   ← 全部推論的根據
```

判斷中點落在哪段，要拿它跟**右端點**比。二分過程保證斷點 `p` 始終落在 `[l, r]` 內，而 `r ≥ p`，所以 `nums[r]` **永遠屬於右段**：

- `nums[m] > nums[r]` → m 在左段（右段的值全都 `≤ nums[r]`）→ 答案嚴格在 m 右邊
- `nums[m] < nums[r]` → m 在右段 → 答案是 m 本身或在 m 左邊

> [!important] 為什麼不能跟 `nums[l]` 比 —— 兩個獨立的理由
> **一、語意會翻。** `l ≤ p` 但有可能**等於** `p`，所以 `nums[l]` 可能在左段、也可能已經是右段的第一個。具體表現就是歧義：`[0,1,2,3]`（沒旋轉）和 `[2,3,0,1]`（有旋轉）都滿足 `nums[m] > nums[l]`，答案卻一個在左、一個在右。`nums[r]` 的段別則永遠穩定。
>
> **二、`m` 可能等於 `l`，變成自己跟自己比。** `m = l + (r-l)/2` 下取整的範圍是 `[l, r-1]` —— **可以等於 `l`，但永遠不等於 `r`**。當 `r == l+1` 時 `nums[m] >= nums[l]` 退化成恆真，無條件往右走：
>
> ```txt
> [2,3,0,1]：
>   l=0 r=3 m=1  a[1]=3 >= a[0]=2  -> l=2
>   l=2 r=3 m=2  a[2]=0 >= a[2]=0  ← 自比！恆真 -> l=3
>   l=3 r=3      回傳 a[3] = 1     ✗ 應為 0
> ```
>
> 實測無保護的左端點比較，105 組窮舉裡失敗 42 組。比 `nums[r]` 天生免疫這個坑。

> [!note] 題目的 constraint 幫不上忙
> 「rotated between `1` and `n` times」**不排除已排序的情況** —— 旋轉 n 次就轉回原狀，題目自己的範例 `[0,1,2,4,5,6,7]` is rotated `7` times 就是。`unique` 那條倒是有用：它保證不會出現 `nums[m] == nums[r]`，嚴格比較才安全（[[0154-Find-Minimum-in-Rotated-Sorted-Array-II]] 沒這條，就得退化成 O(n)）。

### 方法一：比右端點，收縮到剩一個 — O(log n)／O(1)（推薦）

沒有 early return、沒有特判，`if/else` 涵蓋所有情況。不變量是「**最小值的 index 永遠留在 `[l, r]` 內**」，區間每輪至少縮小 1，最後收成單一元素，那就是答案。

```cpp
// Time: O(log n)
// Space: O(1)
class Solution {
 public:
  int findMin(vector<int>& nums) {
    int l = 0, r = nums.size() - 1;
    while (l < r) {
      int m = l + (r - l) / 2;
      if (nums[m] > nums[r]) l = m + 1;  // m 在左段，答案在右
      else r = m;                        // m 在右段，答案是 m 或更左
    }
    return nums[l];
  }
};
```

**為什麼保證收斂**：`l < r` 時 `r - l ≥ 1`，下取整後必有 `l ≤ m < r`。走 `l = m + 1` 則 l 嚴格變大，走 `r = m` 則因 `m < r` 而 r 嚴格變小 —— 兩邊都不可能原地踏步。

> [!warning] 這裡寫 `r = m - 1` 會把答案丟掉
> `nums[m] < nums[r]` 只說明 m 落在右段，**而右段的第一個就是答案**，m 完全可能正是它。實測把這行改成 `r = m - 1`，窮舉長度 1~12 的所有旋轉共 78 組會答錯 24 組（不會死迴圈，就是安靜地給錯答案）。詳細的模板規則見 [[Binary-Search-Templates]]。

### 方法二：先判斷區間是否已排序 — O(log n)／O(1)

另一種直覺是：一旦當前區間本身已經遞增，`nums[l]` 就是答案，直接收工。依據是這個引理 —— 若斷點落在 `(l, r]` 內，則 `nums[l]` 屬左段、`nums[r]` 屬右段，必有 `nums[l] > nums[r]`。取逆否即得：

$$\text{nums}[l] < \text{nums}[r] \iff [l..r] \text{ 內沒有斷點}$$

```cpp
// Time: O(log n)
// Space: O(1)
class Solution {
 public:
  int findMin(vector<int>& nums) {
    int l = 0, r = nums.size() - 1;
    while (true) {
      if (nums[l] <= nums[r]) return nums[l];  // 區間已排序，左端就是答案
      int m = l + (r - l) / 2;
      if (nums[m] >= nums[l]) l = m + 1;       // m 在左段，答案在右
      else r = m;                              // m 可能就是答案，不能 m - 1
    }
  }
};
```

那個每輪重驗的 `if` 同時擋掉了上面兩個坑（區間已排序時直接收工，也就不會走到自比那步）。這是 [[0033-Search-in-Rotated-Sorted-Array]] 的必要手法（那題非得先分辨哪一半有序不可），但用在本題不划算：20 萬組隨機測資（n ≤ 1000）平均輪數只從 8.59 降到 8.01，省不到一輪。

> [!warning] 這裡的 `<=` 是被 `while (true)` 逼出來的，且到 154 會爆
> 因為沒有 `l < r` 當出口，只能靠 `l == r` 時 `nums[l] <= nums[r]` 成立來收尾，所以非寫 `=` 不可。但一旦允許重複值（[[0154-Find-Minimum-in-Rotated-Sorted-Array-II]]），`[3,3,1,3]` 會在 `3 <= 3` 時直接回傳 `3`，正解是 `1`。方法一的 `while (l < r)` 讓終止條件是**結構性的**、不依賴值的比較，就沒有這個包袱。

### 方法三：錨定固定的 `nums[0]`，半開區間 — O(log n)／O(1)

真的想比左邊也可以，關鍵是別跟**移動中的** `nums[l]` 比，改跟**永不改變的** `nums[0]` 比。這樣述詞 `P(i) = nums[i] < nums[0]` 在整個陣列上單調（左段全 false、右段全 true），就是標準的「找第一個 true」。參考點固定，比較的語意從頭到尾不變，所以不需要每輪重驗。

```cpp
// Time: O(log n)
// Space: O(1)
class Solution {
 public:
  int findMin(vector<int>& nums) {
    int n = nums.size();
    int l = 0, r = n;                    // 半開區間，r 是界標不是候選
    while (l < r) {
      int m = l + (r - l) / 2;
      if (nums[m] < nums[0]) r = m;      // m 在右段，邊界 ≤ m
      else l = m + 1;                    // m 在左段，邊界 > m
    }
    return l == n ? nums[0] : nums[l];   // l == n 代表沒有斷點
  }
};
```

> [!tip] 這版為什麼能用半開區間，方法一不行
> 半開區間的 `r` 是**界標不是元素**，所以 `r = n` 合法 —— 但代價是 `nums[r]` 不可解參考。方法一的 `nums[m] > nums[r]` 讀了 `nums[r]`，被迫用閉區間 `r = n - 1`；本版只跟固定的 `nums[0]` 比，從不碰 `nums[r]`，才能用半開。
>
> 好處是 `l == n` 這個狀態**免費表達了「找不到斷點」**，省掉閉區間版必須另外寫的那行 `if (nums[0] <= nums[n-1]) return nums[0];`。慣例的完整比較見 [[Binary-Search-Templates]]。

三種寫法實測（窮舉長度 1~14 全部旋轉 + 隨機 20 萬組含負數與不連續值）全數通過，平均輪數方法一 ≈ 方法三 8.55 < 方法二 8.60。**方法一仍是首選**：最短、沒有特判、不需要額外的錨點概念。

## Related Problems

[[Binary-Search-Templates]] — 本題屬「模板二 · 閉區間」，因為要讀 `nums[r]`；模板選擇、取整配對、區間慣例的完整規則都在這篇
[[0033-Search-in-Rotated-Sorted-Array]] — 同一種旋轉陣列結構，但因為是找 target 而改用模板一，最好的對照組
[[0154-Find-Minimum-in-Rotated-Sorted-Array-II]] — 允許重複值的版本，`nums[m] == nums[r]` 時只能 `r--`，最壞退化成 O(n)
[[0162-Find-Peak-Element]] — 同為模板二找邊界，一樣沒有明確出口、一樣寫 `r = m`
[[0035-Search-Insert-Position]] — 模板二最乾淨的入門題，找第一個 `>= target` 的位置
[[0034-Find-First-and-Last-Position-of-Element-in-Sorted-Array]] — 左右邊界各跑一次模板二，練習兩種取整配對
