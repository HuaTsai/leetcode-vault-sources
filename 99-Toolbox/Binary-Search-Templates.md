---
tags:
  - binary-search
  - pattern
dg-publish: true
---

## 這篇在解決什麼

二分搜尋的概念一句話就講完，難的全在邊界：`while` 要寫 `<` 還是 `<=`？`r` 要更新成 `m` 還是 `m - 1`？初始 `r` 是 `n` 還是 `n - 1`？取整要不要 `+1`？

這些不是要背的，它們**兩兩互相決定**。只要先回答兩個問題，剩下全部推得出來。

## 決策流程

```txt
問題一：能不能寫出一個 if，當場 return m？
  ├─ 能  → 模板一（找 target）    → l = m+1 ／ r = m-1 ／ while (l <= r)
  └─ 不能 → 模板二（找邊界）      → 繼續問題二
                                     ↓
問題二：找「第一個」還是「最後一個」滿足條件的位置？
  ├─ 第一個 → r = m ／ l = m+1 → m 用【下取整】
  └─ 最後一個 → l = m ／ r = m-1 → m 用【上取整】
```

> [!important] 決定一切的是不變量，不是收縮
> 直覺上會以為關鍵是「每輪都要讓區間變小」，但收縮只是**必要條件**。真正的約束是：**答案必須永遠留在區間內**。
>
> `r = m - 1` 收得比 `r = m` 更兇，可是你得先有資格丟掉 `m` —— 也就是先證明 `m` 不可能是答案。模板一的 `if (nums[m] == target) return m;` 就是那張授權書；模板二沒有這種 if，所以丟不得。

## 模板一：找 target — 閉區間 `[l, r]`

```cpp
// Time: O(log n)
// Space: O(1)
int searchTarget(vector<int>& nums, int target) {
  int l = 0, r = nums.size() - 1;
  while (l <= r) {
    int m = l + (r - l) / 2;
    if (nums[m] == target) return m;
    if (nums[m] < target) l = m + 1;
    else r = m - 1;                   // 已證 nums[m] != target，丟得掉
  }
  return -1;                          // 區間變空 = 找不到
}
```

出口是**區間變空**（`l > r`）。`while` 必須寫 `l <= r`，因為 `l == r` 時區間還有一個元素待檢查。

> [!warning] 模板一非 `r = m - 1` 不可
> 它的「找不到」狀態靠 `l > r` 表達，而 `r = m` 在 `l == m == r` 時原地不動，**永遠製造不出 `l > r`**。實測把模板一的 `r = m - 1` 改成 `r = m`：130 組測資裡死迴圈 55 組、答錯 0 組 —— 多留一個候選對正確性永遠安全，對終止性不安全。

## 模板二：找邊界 — 閉區間 `[l, r]`

沒有「當場 return」的條件，只能一路收縮到剩下一個。**答案保證存在**時用這個版本。

```cpp
// Time: O(log n)
// Space: O(1)
int firstAtLeast(vector<int>& nums, int target) {  // 第一個 >= target 的 index
  int l = 0, r = nums.size() - 1;
  while (l < r) {
    int m = l + (r - l) / 2;          // 下取整 -> 保證 l <= m < r
    if (nums[m] >= target) r = m;     // m 可能就是答案，保留
    else l = m + 1;
  }
  return l;                           // 區間剩 1，就是答案
}
```

`while` 寫 `l < r`：`l == r` 代表只剩一個候選、已經是答案，不需要再進迴圈。

> [!warning] 這裡寫 `r = m - 1` 會安靜地給錯答案
> `nums[m] >= target` 只說明 m 滿足條件，**而我們要的正是第一個滿足條件的位置**，m 完全可能就是它。實測在 [[0153-Find-Minimum-in-Rotated-Sorted-Array]] 上把 `r = m` 改成 `r = m - 1`：78 組測資裡死迴圈 0 組、**答錯 24 組**。
>
> 跟模板一的失敗模式剛好鏡像對稱：**丟掉 m 對終止性永遠安全、對正確性不安全；保留 m 反之。**

### 找「最後一個」的鏡像版本

```cpp
// Time: O(log n)
// Space: O(1)
int lastAtMost(vector<int>& nums, int target) {  // 最後一個 <= target 的 index
  int l = 0, r = nums.size() - 1;
  while (l < r) {
    int m = l + (r - l + 1) / 2;      // 上取整 -> 保證 l < m <= r
    if (nums[m] <= target) l = m;     // m 可能就是答案，保留
    else r = m - 1;
  }
  return l;
}
```

> [!note] 取整方式必須跟更新規則配對
> 模板二有一邊不動 ±1，全靠取整保證中點不會撞到那一端：
>
> - `r = m` 配**下取整** —— `l + (r-l)/2` 保證 `m < r`，`r = m` 才會嚴格變小
> - `l = m` 配**上取整** —— `l + (r-l+1)/2` 保證 `m > l`，`l = m` 才會嚴格變大
>
> 配錯就原地踏步。實測 `lastAtMost` 誤用下取整，光是 `[1, 2]` 找最後一個 `<= 2` 就死迴圈。
>
> **模板一則對取整免疫**，因為它兩邊都 ±1，怎麼取整都嚴格收縮（實測改成上取整照樣全過）。

## 區間慣例：`[l, r]` vs `[l, r)`

上面全用閉區間。另一套是半開區間，差別在 `r` 的身分：

|                    | 閉區間 `[l, r]`      | 半開區間 `[l, r)`              |
| ------------------ | -------------------- | ------------------------------ |
| 候選範圍           | `l` 到 `r`（含兩端） | `l` 到 `r-1`，**`r` 不是候選** |
| 初始涵蓋全陣列     | `l = 0, r = n - 1`   | `l = 0, r = n`                 |
| 「空」的表示       | `l > r`              | `l == r`                       |
| `r` 的身分         | 一個**元素**的 index | 一個**界標**，可以是 `n`       |
| `nums[r]` 可否讀取 | 可以                 | **不可以**，會越界             |

`r = n` 之所以合法，正因為半開的 `r` 不是元素而是邊界 —— `[0, n)` 涵蓋 `0..n-1`，完全不需要 `nums[n]` 存在。STL 的 `lower_bound` / `upper_bound` / `partition_point` 全是這個慣例，回傳 `last` 就代表找不到。

```cpp
// Time: O(log n)
// Space: O(1)
int firstAtLeast(vector<int>& nums, int target) {  // 半開版
  int l = 0, r = nums.size();       // r = n
  while (l < r) {
    int m = l + (r - l) / 2;
    if (nums[m] >= target) r = m;
    else l = m + 1;
  }
  return l;                          // == n 代表「全部都 < target」
}
```

> [!warning] 同樣寫 `r = m`，兩種慣例下語意**相反**
>
> ```txt
> 閉區間  [l, r]，r = m  ->  新區間 [l, m]   含 m，保留
> 半開區間 [l, r)，r = m  ->  新區間 [l, m)   不含 m，排除
> ```
>
> 所以「`r = m` 保留、`r = m - 1` 排除」那條規則**只在閉區間成立**。半開下 `r = m` 本身就是排除，再減 1 等於多丟一格。實測半開區間誤寫 `r = m - 1`，105 組裡答錯 42 組。
>
> **兩套慣例絕不能混用**，這是二分 off-by-one 的頭號來源。

### 該選哪一種

> [!tip] 判斷法：迴圈內有沒有解參考 `nums[r]`？
>
> - **有** → 被迫用閉區間，因為 `r` 必須是合法 index。[[0153-Find-Minimum-in-Rotated-Sorted-Array]] 的標準解要比 `nums[m] > nums[r]`，就屬這類。
> - **沒有**（只跟外部固定值比）→ 兩種都行，**半開通常更好**：`l == n` 這個狀態免費幫你表達「找不到」，省掉一次特判。

## 中點的寫法：`l + (r - l) / 2` 與 `std::midpoint`

上面所有模板都寫 `l + (r - l) / 2`。C++20 的 `std::midpoint(l, r)`（`<numeric>`）是同一件事的具名版本，兩者可以互換。

> [!tip] 實測等價，而且是真的零成本
> **語義一致**：`std::midpoint(a, b)` 對整數是「往 `a` 的方向取整」，所以 `l <= r` 時 `midpoint(l, r)` 就是下取整。200 萬組隨機測資（含負數、跨零）比對 `l + (r - l) / 2`，不一致 0 組。
>
> **效能一致**：單看函式，`midpoint` 多一組分支和 `imul`（要決定往哪邊取整）；但放進二分迴圈後，編譯器從迴圈條件 `l <= r` 就證明了方向，那組分支被完全消掉：
>
> ```txt
> l + (r - l) / 2         std::midpoint(l, r)
> mov eax, esi            mov eax, esi
> sub eax, ecx            sub eax, ecx
> sar eax        ← 差這   shr eax        ← 差這
> add eax, ecx            add eax, ecx
> ```
>
> `-O2` 下 cachegrind 實測三種迴圈型態（模板一、模板二下取整、模板二上取整），3.1 億條指令裡差 3 條（程式啟動雜訊），checksum 全部相同。

差異只有一個，而且刷題幾乎踩不到：

> [!warning] `l + (r - l) / 2` 的前提是 `r - l` 自己不溢位
> 這個寫法閃掉的是 `l + r` 溢位，卻沒有保證 `r - l` 安全。當區間橫跨整個 int 值域：
>
> ```txt
> l = -2000000000, r = 2000000000
>   std::midpoint(l, r) = 0             正確
>   l + (r - l) / 2     = -2147483648   錯，且 r - l 已是 signed overflow（UB）
> ```
>
> index 二分（`l >= 0`）和 [[0875-Koko-Eating-Bananas]] 那種正值域二分都不可能觸發，所以本篇模板維持原寫法。只有**在橫跨正負的值域上二分**才有實際差別。

> [!note] 兩個使用限制
>
> - **吃不了 iterator**：只支援算術型別和原生指標，`vector<int>::iterator` 直接編譯失敗（`no matching function for call to midpoint`）。`int*`、`size_t`、`long long` 都可以。
> - **上取整要靠反轉參數**：往第一個參數取整，所以 `midpoint(r, l)` 才是上取整。實測正確、也一樣零成本，但它跟 `midpoint(l, r)` 長得太像，review 時容易看漏 —— **建議上取整仍寫 `l + (r - l + 1) / 2`**，把「+1 是為了上取整」這個意圖留在字面上。

面試時兩種寫法怎麼講、會被追問什麼，見 [[Interview-English-Phrasebook]]。

## 題目分類

| 模板                       | 題目                                                                                                                                                                                                     |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 模板一（找 target）        | [[0704-Binary-Search]]、[[0033-Search-in-Rotated-Sorted-Array]]、[[0074-Search-a-2D-Matrix]]                                                                                                             |
| 模板二（找邊界）           | [[0035-Search-Insert-Position]]、[[0034-Find-First-and-Last-Position-of-Element-in-Sorted-Array]]、[[0153-Find-Minimum-in-Rotated-Sorted-Array]]、[[0162-Find-Peak-Element]]、[[0278-First-Bad-Version]] |
| 模板二（在答案值域上二分） | [[0875-Koko-Eating-Bananas]]、[[1011-Capacity-To-Ship-Packages-Within-D-Days]]、[[0410-Split-Array-Largest-Sum]]                                                                                         |

最後一類特別值得注意：二分的對象**不是陣列 index，而是答案本身的值域**。述詞是「這個答案值可行嗎」，可行性單調（可行的都可行、不可行的都不可行），所以照樣套模板二找第一個可行值。

> [!tip] 0033 和 0153 是最好的對照組
> 同一種旋轉陣列結構，卻分屬兩個模板 —— 差別**只在你要找什麼**。找 target 有明確出口 → 模板一；找最小值是找邊界 → 模板二。這說明模板的選擇跟資料長什麼樣無關，只跟問題形態有關。

## Related Problems

[[0153-Find-Minimum-in-Rotated-Sorted-Array]] — 模板二 · 閉區間的代表，因為要讀 `nums[r]` 而不能用半開
[[0033-Search-in-Rotated-Sorted-Array]] — 模板一的代表，跟 0153 同結構不同模板
[[0704-Binary-Search]] — 模板一最乾淨的原型
[[0035-Search-Insert-Position]] — 模板二最乾淨的原型
[[0875-Koko-Eating-Bananas]] — 在答案值域上二分的入門題
