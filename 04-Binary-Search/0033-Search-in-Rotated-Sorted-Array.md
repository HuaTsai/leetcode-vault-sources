---
leetcode-id: 33
difficulty: medium
tags:
  - binary-search
  - array
  - grind-169
  - neetcode-150
memo: 拿中點跟右端點比判斷它落在哪一段，有序那段用值域夾擠 target；易錯在三分法沒窮盡，漏掉中點值等於右端點值（即 m ＝ r 那格）就死迴圈，而 target 等於右端點時必須併進向右那一支
dg-publish: true
---

## Problem Description

There is an integer array `nums` sorted in ascending order (with distinct values).

Prior to being passed to your function, `nums` is possibly left rotated at an unknown index `k` (`1 <= k < nums.length`) such that the resulting array is `[nums[k], nums[k+1], ..., nums[n-1], nums[0], nums[1], ..., nums[k-1]]` (0-indexed). For example, `[0,1,2,4,5,6,7]` might be left rotated by `3` indices and become `[4,5,6,7,0,1,2]`.

Given the array `nums` after the possible rotation and an integer `target`, return the index of `target` if it is in `nums`, or `-1` if it is not in `nums`.

You must write an algorithm with `O(log n)` runtime complexity.

Constraints:

- All values of `nums` are unique.

## Solution

核心觀念：旋轉後的陣列是**兩段各自遞增、且左段每個元素都大於右段每個元素**（跟 [[0153-Find-Minimum-in-Rotated-Sorted-Array]] 同一個結構）。二分法本身要求全域單調，這裡不成立，所以每一輪得多做一件事：**先判斷中點落在哪一段，再用那一段的值域夾擠 target**。

```txt
   4  5  6  7 │ 0  1  2        p ＝ 4（最小值下標 ＝ 旋轉點）
   └─左段(大)─┘ └右段(小)┘

   左段 nums[l..p-1]：遞增，值全部 > nums[r]
   右段 nums[p..r]  ：遞增，值全部 ≤ nums[r]   ← nums[r] 是右段的最大值

   所以只要比 nums[m] 和 nums[r]，就知道 m 落在哪一段：
     nums[m] ≤ nums[r] → m 在右段
     nums[m] >  nums[r] → m 在左段
```

> [!important] 為什麼「判斷落在哪一段」就足夠
> 定位出 m 的段別後，target 的所有可能位置都被 `nums[m]` 和 `nums[r]` 這兩個值切乾淨了。以 m 在右段為例：`nums[m] < target ≤ nums[r]` 時 target 只能在 `[m+1, r]`；否則 target 要嘛在右段的更左邊 `[p, m-1]`、要嘛整個在左段 `[l, p-1]` —— 兩者都落在 `[l, m-1]` 內，所以合併成同一個動作 `r = m-1`。
>
> 關鍵在於 `p` 只出現在**推理**裡，不出現在 code 裡。你不需要真的知道 `p` 在哪，只需要知道「兩段的值域不重疊」這件事。

### 方法一：比右端點 — O(log n)／O(1)（推薦）

完整的情況分析。`p` 是最小值下標，僅用於推導：

```txt
0. nums[m] == target                          → return m
1. nums[m] ≤ nums[r]（m 在右段，含 m == r）
     nums[m] < target ≤ nums[r] → target ∈ [m+1, r]                → l = m+1
     否則                         → target ∈ [p, m-1] 或 [l, p-1]   → r = m-1
2. nums[m] > nums[r]（m 在左段）
     nums[r] < target < nums[m]  → target ∈ [l, m-1]               → r = m-1
     否則                         → target ∈ [m+1, p-1] 或 [p, r]   → l = m+1
```

```cpp
// Time: O(log n)
// Space: O(1)
class Solution {
 public:
  int search(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l <= r) {
      int m = l + (r - l) / 2;
      if (nums[m] == target) return m;
      if (nums[m] <= nums[r]) {                                 // m 在右段（含 m == r）
        if (nums[m] < target && target <= nums[r]) l = m + 1;
        else r = m - 1;
      } else {                                                  // m 在左段
        if (nums[r] < target && target < nums[m]) r = m - 1;
        else l = m + 1;
      }
    }
    return -1;
  }
};
```

有 `nums[m] == target` 這張授權書，所以兩邊都能安全地 `±1`，屬於 [[Binary-Search-Templates]] 的模板一。出口是區間變空 `l > r`。

> [!warning] 三分法必須窮盡，否則死迴圈
> 直覺會寫成 `if (nums[m] < nums[r]) ... else if (nums[m] > nums[r]) ...`，但這是**三分法只列了兩支**。值互異，所以 `nums[m] == nums[r]` 只可能是 `m == r`，也就是區間收到剩最後一格——那一格不進任何分支，`l`、`r` 原地不動，直接卡死。連 `nums = [1]` 找 `0` 都會卡。實測 1456 組裡**死迴圈 504 組**。
>
> 該併進情況 1：`m == r` 時 `[m, r]` 是退化的單元素段、trivially 有序，套情況 1 的規則會得到 `nums[m] < target && target <= nums[m]` 恆假 → `r = m - 1` → 區間正確清空。所以寫成 `<=` 就自動吸收，不必特判。
>
> 通則：**只要比較對象是陣列內的另一個元素（而非外部固定值），`==` 就一定會在 `l == r` 時發生**，分支必須留 `else` 收尾。

> [!warning] `target <= nums[r]` 的等號不能省，但另一支不能加
> 目前只排除了 `target != nums[m]`，`target == nums[r]` 完全可能發生——而它就在 index `r`，兩種情況下**正解都是 `l = m + 1`**（`r` 永遠在右半）。但兩支把「向右」放在條件的相反側：
>
> ```txt
> 情況 1： if  成立 → l = m+1   ← 向右在 if，等號必須寫進條件
> 情況 2： else 成立 → l = m+1   ← 向右在 else，等號白撿
> ```
>
> 所以最終 code 裡兩支的等號不對稱不是筆誤，是推導的必然。實測情況 1 寫成嚴格的 `target < nums[r]`：**答錯 177 組**（`[1,3]` 找 `3` 就回 `-1`），不會死迴圈，安靜地錯。

### 方法二：判斷左半是否有序 — O(log n)／O(1)

教科書最常見的形式。改問「`[l, m]` 這一段有沒有被斷點切開」，不去定位段別：

```cpp
// Time: O(log n)
// Space: O(1)
class Solution {
 public:
  int search(vector<int>& nums, int target) {
    int l = 0, r = nums.size() - 1;
    while (l <= r) {
      int m = l + (r - l) / 2;
      if (nums[m] == target) return m;
      if (nums[l] <= nums[m]) {                                 // [l, m] 有序
        if (nums[l] <= target && target < nums[m]) r = m - 1;
        else l = m + 1;
      } else {                                                  // [m, r] 有序
        if (nums[m] < target && target <= nums[r]) l = m + 1;
        else r = m - 1;
      }
    }
    return -1;
  }
};
```

> [!tip] 為什麼這題可以跟 `nums[l]` 比，0153 卻不行
> [[0153-Find-Minimum-in-Rotated-Sorted-Array]] 禁止跟 `nums[l]` 比，是因為那題要**定位斷點相對於 m 的方向**，而 `nums[l]` 的段別本身有歧義（`l` 可能已經等於 `p`）。
>
> 本題的述詞不一樣：`nums[l] <= nums[m]` ⟺ **`[l, m]` 內沒有斷點**。這是個自足的等價命題，證明只要兩行——沒斷點則該段遞增故 `nums[l] < nums[m]`；有斷點則 `nums[l]` 在左段、`nums[m]` 在右段故 `nums[l] > nums[m]`。它從頭到尾不需要知道 `p` 在哪，所以沒有歧義問題。
>
> **兩題的差別在問題形態，不在資料長相。**

> [!warning] `nums[l] <= nums[m]` 的等號同樣不能省
> 區間剩兩格時 `m == l`，`nums[l] < nums[m]` 恆假，會誤判成「右半有序」。實測寫成嚴格：**答錯 18 組**，最小反例 `[3,1]` 找 `1` 回 `-1`。

### 方法三：先找斷點，再對虛擬展開的序列二分 — O(log n)／O(1)

另一條路：先用 [[0153-Find-Minimum-in-Rotated-Sorted-Array]] 求出斷點 `p`，之後把 `nums` 當成**虛擬的完整遞增陣列** `A[i] = nums[(i + p) % n]` 來搜。第二段就退化成零判斷的標準二分。

```cpp
// Time: O(log n)
// Space: O(1)
class Solution {
 public:
  int search(vector<int>& nums, int target) {
    int n = nums.size();
    int l = 0, r = n - 1;
    while (l < r) {                        // 模板二：找最小值下標
      int m = l + (r - l) / 2;
      if (nums[m] > nums[r]) l = m + 1;
      else r = m;
    }
    int p = l;
    l = 0, r = n - 1;
    while (l <= r) {                       // 模板一：在虛擬展開的遞增序列上找 target
      int m = l + (r - l) / 2;
      int i = (m + p) % n;
      if (nums[i] == target) return i;
      if (nums[i] < target) l = m + 1;
      else r = m - 1;
    }
    return -1;
  }
};
```

兩次二分是 `2 · O(log n)`，仍是 `O(log n)`。**不要真的重排陣列**（那會變 O(n)），`% n` 只是座標映射。

> [!warning] `return i` 不是 `return m`
> 這一版憑空造出第二套下標系：`m` 是虛擬遞增序列上的邏輯位置，`i` 才是真實 index。題目要的是真實 index。實測寫成 `return m`：**答錯 572 組**，是三個方法的所有錯版裡最嚴重的。這就是模運算換來的代價——方法一、二全程只碰真實下標，沒有這個坑。

> [!note] 好在哪、不好在哪
> **好**：第二段是完全沒有 case analysis 的模板一，難的部分全部壓進第一段，而第一段就是 0153，可以直接複用已經練熟的東西。
>
> **不好**：多出一套下標系（上面那個坑），而且**無法延伸到 [[0081-Search-in-Rotated-Sorted-Array-II]]**——允許重複值時 `[1,1,1,0,1,1]` 根本沒辦法在 O(log n) 內定位 `p`，二分的前提就沒了。方法一、二只要加一行處理相等的情況就能沿用，最壞退化成 O(n) 但仍然正確。**想長到 81，得走方法一或方法二的形狀。**

三種寫法實測（窮舉長度 1~12 的所有旋轉 × 所有 target，共 1456 組）全數通過。**方法一是首選**：只有一套下標、可以完整推導、且能延伸到 81。

## Related Problems

[[Binary-Search-Templates]] — 本題屬「模板一 · 找 target」，因為有 `nums[m] == target` 這個明確出口；模板選擇與區間慣例的完整規則都在這篇
[[0153-Find-Minimum-in-Rotated-Sorted-Array]] — 同一種旋轉陣列結構，但因為是找邊界而改用模板二，最好的對照組；方法三直接把它當子程序用
[[0081-Search-in-Rotated-Sorted-Array-II]] — 允許重複值的版本，斷點無法二分定位，只有方法一、二的形狀能延伸，最壞退化成 O(n)
[[0704-Binary-Search]] — 模板一最乾淨的原型，方法三的第二段就是它
[[0034-Find-First-and-Last-Position-of-Element-in-Sorted-Array]] — 同樣是「找 target」但要求邊界，展示明確出口消失後為何得換模板二
