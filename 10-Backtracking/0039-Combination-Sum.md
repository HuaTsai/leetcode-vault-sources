---
leetcode-id: 39
difficulty: medium
tags:
  - array
  - backtracking
  - grind-169
  - neetcode-150
memo: 要列舉所有組合就只能回溯，同一題改問幾種就變成完全背包計數 DP；可重複取所以遞迴傳 i 而非 i + 1，start 單調遞增自然去掉排列重複；剪枝必須擋在遞迴之前，排序只是讓它從 continue 升級成 break
dg-publish: true
---

## Problem Description

Given an array of distinct integers `candidates` and a target integer `target`, return a list of all unique combinations of `candidates` where the chosen numbers sum to `target`. You may return the combinations in any order.

The same number may be chosen from `candidates` an unlimited number of times. Two combinations are unique if the frequency of at least one of the chosen numbers is different.

The test cases are generated such that the number of unique combinations that sum up to `target` is less than `150` combinations for the given input.

Constraints:

- All elements of candidates are distinct.

## Solution

核心觀念：題目要的是**列舉所有組合**，不是「有幾種」也不是「最少幾個」，所以輸出量本身就是指數級——這正是回溯的判定條件。剩下兩個設計決定：**可重複取**代表遞迴時 `start` 停在原地不前進，而**組合不論順序**代表 `start` 只能單調遞增，`[2,3]` 和 `[3,2]` 由此天然只會產生一次。

```txt
candidates = [2, 3, 6, 7]（已排序）   target = 7

bt(start=0, remain=7)
├─ 選 2 → bt(0, 5)          start 留在 0，2 還能再選
│  ├─ 選 2 → bt(0, 3)
│  │  ├─ 選 2 → bt(0, 1)    1 < 2，迴圈一次都不跑     ✗
│  │  └─ 選 3 → bt(1, 0)    remain == 0               ✓ [2,2,3]
│  ├─ 選 3 → bt(1, 2)       start=1，不會再回頭撿 2   ✗
│  └─ 6, 7 > 5 → break
├─ 選 3 → bt(1, 4)  ✗
├─ 選 6 → bt(2, 1)  ✗
└─ 選 7 → bt(3, 0)  ✓ [7]
```

> [!important] 看輸出形狀就能判定該用回溯還是 DP
> 同一個「湊出 target」的骨架，問法一換解法就換：
>
> | 問法 | 解法 |
> | --- | --- |
> | 列出**所有**組合 | 回溯（本題） |
> | 有**幾種**組合 | 完全背包計數 DP → [[0518-Coin-Change-II]] |
> | **最少**幾個湊出來 | 完全背包最小化 DP → [[0322-Coin-Change]] |
>
> 差別在於 DP 只能把答案摺疊成一個標量（數量、最優值），摺不了解集合。要求列舉時輸出本身就是指數級，DP 沒有任何壓縮空間可用——方法三會實測驗證這點。

以下三個方法的複雜度共用同一組符號：**`N`** = `candidates.size()`（候選個數）、**`T`** = `target`、**`M`** = `min(candidates)`（最小的候選值）。題目約束下 `N ≤ 30`、`T ≤ 40`、`M ≥ 2`。

### 方法一：for 迴圈 + 排序剪枝 — O(N^(T/M+1))／O(T/M)

每一步至少從 `remain` 扣掉 `M`，所以遞迴深度上限是 `T/M`，而每層最多 `N` 個分支；空間不計輸出，只有遞迴堆疊和 `cur` 這條路徑。

```cpp
// Time: O(N^(T/M+1))  遞迴樹深度 T/M，每層 N 個分支
// Space: O(T/M)       遞迴堆疊 + cur，不計輸出
class Solution {
 public:
  vector<vector<int>> combinationSum(vector<int>& candidates, int target) {
    nums = candidates;
    sort(nums.begin(), nums.end());  // 為了讓下面的迴圈條件能當 break 用
    bt(0, target);
    return ans;
  }

 private:
  void bt(int start, int remain) {
    if (remain == 0) {
      ans.push_back(cur);
      return;
    }
    for (int i = start; i < (int)nums.size() && nums[i] <= remain; ++i) {
      cur.push_back(nums[i]);
      bt(i, remain - nums[i]);  // 傳 i 不是 i + 1，同一個數還能再選
      cur.pop_back();
    }
  }

  vector<int> nums, cur;
  vector<vector<int>> ans;
};
```

這個版本疊了三件**互相獨立**的事，拆開看才不會誤以為它們綁在一起：

1. **用 for 迴圈取代「選／不選」的二元遞迴**（與排序無關）。二元版連「跳過某個數」都要吃掉一層堆疊，深度是 `O(T/M + N)`；for 版把跳過變成 `++i`，深度降到 `O(T/M)`。附帶好處是 `i` 的邊界由迴圈條件守住，不必在呼叫端寫守衛。
2. **剪枝擋在遞迴之前**（與排序無關）。`nums[i] > remain` 就完全不要進那次遞迴。不排序照樣做得到，只是得寫成 `if (nums[i] > remain) continue;`。
3. **排序**。唯一作用是把第 2 點從 `continue` 升級成迴圈終止條件（等同 `break`），省下迴圈空轉的比較。

> [!warning] 排序和 `nums[i] <= remain` 是綁在一起的，拆開會直接漏解
> 迴圈條件寫成 `nums[i] <= remain` 等於宣告「遇到第一個太大的就可以收工」，這個推論只有在**遞增排序**下才成立。忘記 `sort` 的話：
>
> ```cpp
> // candidates = {7, 2, 3}, target = 5
> for (int i = start; i < n && nums[i] <= remain; ++i)  // nums[0] = 7 > 5 → 迴圈當場結束
> ```
>
> 實測輸出 `[]`，正確答案是 `[[2,3]]`。不想排序就老實用 `continue`。

> [!tip] `start` 單調遞增就是這題全部的去重機制
> 因為 `candidates` 保證相異，只要限制「後面選的下標不小於前面」，每個組合就只會以遞增序列的形式被走到一次，不需要任何 `set` 或事後過濾。若候選本身含重複值（[[0040-Combination-Sum-II]]），這招不夠，還要加同層跳過：`if (i > start && nums[i] == nums[i - 1]) continue;`。

### 方法二：選／不選 的二元遞迴 — O(N^(T/M+1))／O(T/M + N)

把每個下標看成一個獨立的二元決策：**選 `nums[i]`（下標不動，之後還能再選它）** 或 **不選（前進到 `i + 1`）**。結構上和 [[0078-Subsets]] 的骨架同源，只是多了 `remain` 當作剪枝與收集的依據。

```cpp
// Time: O(N^(T/M+1))
// Space: O(T/M + N)  「跳過」也佔一層堆疊，深度比方法一多 N
class Solution {
 public:
  vector<vector<int>> combinationSum(vector<int>& candidates, int target) {
    nums = candidates;
    bt(0, target);
    return ans;
  }

 private:
  void bt(int i, int remain) {
    if (remain == 0) {
      ans.push_back(cur);
      return;
    }
    if (remain < 0 || i == (int)nums.size()) return;  // 守衛放入口，任何 i 都能安全呼叫

    cur.push_back(nums[i]);   // 選：下標不動，下一層還能再選同一個數
    bt(i, remain - nums[i]);
    cur.pop_back();

    bt(i + 1, remain);        // 不選：換下一個候選
  }

  vector<int> nums, cur;
  vector<vector<int>> ans;
};
```

> [!important] 邊界檢查要放在函式入口，不是呼叫端
> 常見的寫法是把越界判斷寫在呼叫的那一行：
>
> ```cpp
> if (i + 1 < (int)nums.size()) bt(i + 1, remain);   // 呼叫端守衛
> ```
>
> 這樣 `bt` 的前置條件變成「呼叫者必須保證 `i` 合法」，而函式內部的 `nums[i]` 是**無條件存取**的。讀者要確認不會越界，得回頭掃過每一個 call site。改成入口的 `if (i == (int)nums.size()) return;` 之後，契約才完整：任何 `i ∈ [0, n]` 都可以呼叫，呼叫端什麼都不用想。

> [!note] 參數命名與成員狀態
> 遞迴進去之後那個值已經不是原本的 `target` 了，命名為 `remain` 才對得上語意。另外 `cur` / `ans` 放成員在 LeetCode 沒問題（每個測資新建一次 `Solution`），代價是物件變成有狀態——同一個實例呼叫兩次 `combinationSum`，`ans` 會累積。在意的話就在函式開頭清空，或改成引用參數往下傳。

### 方法三：完全背包 DP 堆出所有組合 — 慢，僅作對照

`dp[t]` 直接存**所有**和為 `t` 的組合。外層跑物品、內層容量遞增，就是完全背包的標準骨架（見 [[Knapsack-and-Classic-DP]]），也保證每個組合只會照 `candidates` 的順序生成一次。

```cpp
// Time / Space: 由所有組合的總長度主導，遠差於回溯
class Solution {
 public:
  vector<vector<int>> combinationSum(vector<int>& candidates, int target) {
    vector<vector<vector<int>>> dp(target + 1);
    dp[0].push_back({});  // 和為 0 只有空組合一種

    for (int c : candidates) {             // 外層物品：組合內元素只會照 candidates 的順序出現
      for (int t = c; t <= target; ++t) {  // 內層容量遞增：同一個 c 可重複取
        for (auto comb : dp[t - c]) {      // 刻意用 by value 複製一份再接上 c
          comb.push_back(c);
          dp[t].push_back(std::move(comb));
        }
      }
    }
    return dp[target];
  }
};
```

> [!note] 實測：列舉題用 DP 沒有好處
> 量測條件：`candidates` = 12 個質數 `{2,3,5,7,11,13,17,19,23,29,31,37}`、`target = 40`（302 組解），`g++ -std=c++20 -O2`，各跑 500～2000 次取平均。「呼叫次數」為 `bt` 的進入次數。
>
> | 版本 | 呼叫次數 | 時間 |
> | --- | --- | --- |
> | 方法二 二元遞迴 | 35,554 | 0.064 ms |
> | for 迴圈，不剪枝 | 18,976 | — |
> | for + `continue` 剪枝（不排序） | 2,699 | 0.026 ms |
> | 方法一 for + `break` 剪枝（有排序） | **2,699** | **0.017 ms** |
> | 方法三 完全背包 DP | — | 0.172 ms |
>
> 兩個結論。其一，**排序對剪枝效果毫無貢獻**：`continue` 版和 `break` 版剪掉的遞迴節點數完全相同（都是 2,699），7 倍的節點縮減全部來自「把判斷提前到遞迴之前」，排序只多省下迴圈空轉的比較，換來約 1.5 倍常數差。其二，**DP 版反而最慢**（比方法一慢約 10 倍），因為每個 `dp[t]` 要實際存下整串 `vector`、複製成本主導了整個過程——這就是「列舉題別想用 DP 壓」的實證。

## Related Problems

- [[0040-Combination-Sum-II]] — 候選含重複值且每個只能用一次，`start` 去重不夠，還要加同層跳過
- [[0518-Coin-Change-II]] — 同一個湊數骨架改問「幾種」，摺成完全背包計數 DP
- [[0322-Coin-Change]] — 同一個骨架改問「最少幾個」，摺成完全背包最小化 DP
- [[0077-Combinations]] — 拿掉「可重複取」的基礎款，同樣靠 `start` 單調遞增保證組合唯一
- [[0078-Subsets]] — 方法二那個「選／不選」二元遞迴骨架的原型
- [[0046-Permutations]] — 對照組：排列要求順序不同也算，所以不能用 `start`，改用 `used` 陣列
- [[Knapsack-and-Classic-DP]] — 完全背包骨架，以及「最優值／方案數／列舉」三種摺疊方式的分界
