---
leetcode-id: 11
difficulty: medium
tags:
  - two-pointers
  - array
  - greedy
  - grind-169
  - neetcode-150
memo: 面積由較矮那條線決定，所以每次只移動較矮的一端，因為移動較高的一端寬變小、高仍被矮邊卡住，面積只會更差
dg-publish: true
---

## Problem Description

You are given an integer array `height` of length `n`. There are `n` vertical lines drawn such that the two endpoints of the `ith` line are `(i, 0)` and `(i, height[i])`.

Find two lines that together with the x-axis form a container, such that the container contains the most water.

Return the maximum amount of water a container can store.

Notice that you may not slant the container.

## Solution

核心觀念：面積 `= (r - l) * min(h[l], h[r])`，由**較矮**那條線封頂。從最寬的兩端 `l=0`、`r=n-1` 開始，每次往內收掉較矮的一邊——犧牲寬度去賭一個更高的瓶頸，指針相遇即止，O(n) 一遍掃完。

```txt
height: [1, 8, 6, 2, 5, 4, 8, 3, 7]
         l                       r     min(1,7)*8=8，左邊矮 → l++
            l                    r     min(8,7)*7=49 ✔，右邊矮 → r--
```

```cpp
// Time: O(n)
// Space: O(1)
class Solution {
 public:
  int maxArea(vector<int>& height) {
    int l = 0, r = height.size() - 1, best = 0;
    while (l < r) {
      best = max(best, (r - l) * min(height[l], height[r]));
      if (height[l] < height[r]) {
        ++l;
      } else {
        --r;
      }
    }
    return best;
  }
};
```

> [!important] 為什麼只移動較矮的一邊？
> 設 `height[l] < height[r]`，面積被矮的 `l` 卡住。往內收一定變窄，只能寄望換到更高的線補回來：
>
> - **移動較高的 `r`**：寬變小，高仍被 `l` 封頂（`min` 不會超過 `height[l]`），面積只會更差 → 沒意義。
> - **移動較矮的 `l`**：寬雖變小，但有機會換到更高的線讓 `min` 變大 → 唯一可能變好的方向。
>
> 收掉矮邊時，丟掉的是「以這條矮線為瓶頸」裡最寬的配對，後面更窄的只會更小，安全捨棄、不漏最優解。

### 正確性證明（迴圈不變量）

把每對 `(i, j)`（`i < j`）的面積記為 `A(i, j) = (j − i) · min(h[i], h[j])`，目標是 `OPT = max{ A(i, j) : 0 ≤ i < j ≤ n−1 }`。演算法維護視窗 `[l, r]`（初始 `[0, n−1]`）與 `best`：每步先把 `A(l, r)` 併進 `best`，再把較矮的一端往內收。定義視窗內最佳值 `M(l, r) = max{ A(i, j) : l ≤ i < j ≤ r }`（`l ≥ r` 時記為 −∞）。

**不變量**：每次迭代開始時 `max(best, M(l, r)) = OPT`。

- **初始**：`best = 0`、視窗 `[0, n−1]` 含全部配對，故 `M(0, n−1) = OPT`，而 `max(0, OPT) = OPT`（面積非負），不變量成立。

**關鍵引理（矮端封頂）**：設 `h[l] ≤ h[r]`（即將收掉 `l`）。則對所有 `l < k ≤ r`：

```txt
寬： k − l ≤ r − l
高： min(h[l], h[k]) ≤ h[l] = min(h[l], h[r])
⇒  A(l, k) = (k − l)·min(h[l], h[k]) ≤ (r − l)·h[l] = A(l, r)
```

亦即「所有用到端點 `l` 的配對」皆 ≤ 剛算過的 `A(l, r)`。

**歸納步**：由 `h[l] ≤ h[r]` 收掉 `l`，視窗變 `[l+1, r]`、`best' = max(best, A(l, r))`。視窗 `[l, r]` 的配對恰分兩類——用到 `l` 的 `(l, k)`、與不用 `l` 的（即 `M(l+1, r)`）——故

```txt
M(l, r) = max( max_{l<k≤r} A(l, k),  M(l+1, r) )
        = max( A(l, r),              M(l+1, r) )   （前一項上界即引理的 A(l, r)）
```

代回不變量：`OPT = max(best, M(l, r)) = max(best, A(l, r), M(l+1, r)) = max(best', M(l+1, r))`，故不變量在 `[l+1, r]` 仍成立。`h[l] > h[r]`（含相等時程式收 `r`）的情形對稱可證。

**終止**：每步 `r − l` 減 1，至多 `n−1` 步後 `l = r`，視窗空、`M = −∞`，不變量給出 `best = max(best, −∞) = OPT`。故回傳的 `best` 等於 `OPT`。∎

> [!tip] 為什麼這個不變量對應你的直覺
> 視窗 `[l, r]` 從 `[0, n−1]`（全部配對）開始、每步只收縮，而不變量保證它**始終夾著最優解**（或最優解已落袋為 `best`）。所以「往內收會不會錯過更寬又更好的」不可能發生：引理證明了每次被收掉的那一端，其剩餘配對全被 `A(l, r)` 支配。

## Related Problems

[[0042-Trapping-Rain-Water]] — 同樣兩端雙指針，但收的是每一格的積水，當前格水量由較矮的一側邊界決定
[[0015-3Sum]] — 另一種相向雙指針，靠排序後的單調性決定收哪一邊
