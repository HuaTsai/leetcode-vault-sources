---
leetcode-id: 40
difficulty: medium
tags:
  - array
  - backtracking
  - neetcode-150
memo: 候選含重複值且每個只能用一次，回溯時遞迴傳 `i + 1`，但 `start` 單調遞增不再足以去重，必須排序後在同一層跳過相同值；條件是 `i > start` 而非 `i > 0`，因為 `i == start` 時相等的前一個是上層選走的，跳掉會漏掉同一個值取兩次的合法解；去重一定要發生在搜尋當下，事後用 set 過濾會多走近千倍節點
dg-publish: true
---

## Problem Description

Given a collection of candidate numbers (`candidates`) and a target number (`target`), find all unique combinations in `candidates` where the candidate numbers sum to `target`.

Each number in `candidates` may only be used once in the combination.

Note: The solution set must not contain duplicate combinations.

Constraints:

- 1 <= `candidates[i]` <= 50

## Solution

核心觀念：骨架和 [[0039-Combination-Sum]] 完全一樣，只換了兩個條件——**每個數只能用一次**，所以遞迴傳 `i + 1` 而不是 `i`；**候選含重複值**，所以 `start` 單調遞增不再保證組合唯一。全部的難點都集中在第二點：排序後重複值相鄰，同一層迴圈會拿下標不同、值卻相同的候選各展開一次，產出一模一樣的組合。

```txt
candidates = [10, 1, 2, 7, 6, 1, 5]   target = 8
排序後       [ 1, 1, 2, 5, 6, 7, 10]
               ↑    ↑
              i=0  i=1        同一層 bt(start=0) 的兩個分支

├─ i=0 選 nums[0]=1 → bt(1, 7)   往後可用 [1, 2, 5, 6, 7, 10]
│                                 → [1,1,6] [1,2,5] [1,7]
└─ i=1 選 nums[1]=1 → bt(2, 7)   往後可用 [2, 5, 6, 7, 10]
                                  → [1,2,5] [1,7]        ← 一條新的都沒有
```

> [!important] 同層的第二個重複值一定產不出新解
> 兩個分支往 `cur` 放的值一模一樣，差別只在往後能搜的範圍：`i=1` 分支的 `[2, n)` 是 `i=0` 分支 `[1, n)` 的**真子集**。子集搜出來的組合必然是超集搜出來的子集合，所以後者只會生重複的。推論出去就是去重規則——**同一層裡，一串相同的值只讓第一個當代表**。

以下兩個方法的複雜度共用符號：**`N`** = `candidates.size()`、**`K`** = 相異值的個數。最壞情形（候選全相異）兩者都退化成「每個元素選／不選」的 `2^N` 棵樹，且每找到一組解要複製長度 `O(N)` 的 `cur`。

### 方法一：排序後同層跳過重複值 — O(N·2^N)／O(N)

```cpp
// Time: O(N·2^N)  最壞每個元素選／不選，每組解額外複製 O(N)
// Space: O(N)     遞迴堆疊 + cur，不計輸出
class Solution {
 public:
  vector<vector<int>> combinationSum2(vector<int>& candidates, int target) {
    nums = candidates;
    ranges::sort(nums);  // 讓重複值相鄰，同時讓 nums[i] <= remain 能當終止條件
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
      if (i > start && nums[i] == nums[i - 1]) continue;  // 同層只讓第一個代表
      cur.push_back(nums[i]);
      bt(i + 1, remain - nums[i]);  // i + 1：每個下標只能用一次
      cur.pop_back();
    }
  }

  vector<int> nums, cur;
  vector<vector<int>> ans;
};
```

> [!warning] 是 `i > start`，不是 `i > 0`
> 這是本題唯一的陷阱，兩者的語意天差地別：
>
> | 條件 | `nums[i - 1]` 的身分 | 該不該跳 |
> | --- | --- | --- |
> | `i > start` | 本層剛試過又 `pop_back` 撤銷的兄弟 | 跳，否則整棵子樹重複 |
> | `i == start` | 上一層選走的，**還留在 `cur` 裡** | 不能跳，這是「同一個值取兩個」 |
>
> 寫成 `i > 0` 會把後者一起砍掉。實測 `candidates = [10,1,2,7,6,1,5]`、`target = 8`，正確答案 4 組，`i > 0` 版只剩 3 組——`[1,1,6]` 不見了。

### 方法二：壓成 (值, 次數) 後決定取幾個 — O(N·2^N)／O(N)

先把排序後的陣列摺成 `(值, 出現次數)`，每層只處理一種值、決定取 0…`cnt` 個。去重從「執行時跳過」升級成「結構上不可能重複」，因為同一個值在整棵樹裡只會被決策一次。

```cpp
// Time: O(N·2^N)  上界同方法一；重複值越多，實際走的節點越少
// Space: O(N)     items O(K) + cur O(N) + 遞迴深度 O(K)
class Solution {
 public:
  vector<vector<int>> combinationSum2(vector<int>& candidates, int target) {
    vector<int> nums = candidates;
    ranges::sort(nums);
    for (int x : nums) {
      if (!items.empty() && items.back().first == x) ++items.back().second;
      else items.push_back({x, 1});
    }
    bt(0, target);
    return ans;
  }

 private:
  void bt(int idx, int remain) {
    if (remain == 0) {
      ans.push_back(cur);
      return;
    }
    if (idx == (int)items.size() || items[idx].first > remain) return;

    auto [v, cnt] = items[idx];
    bt(idx + 1, remain);  // 這個值取 0 個

    int take = 0;
    while (take < cnt && v * (take + 1) <= remain) {
      ++take;
      cur.push_back(v);
      bt(idx + 1, remain - v * take);  // 取 take 個
    }
    cur.resize(cur.size() - take);  // 一次撤銷本層放進去的 take 個
  }

  vector<pair<int, int>> items;  // (值, 出現次數)
  vector<int> cur;
  vector<vector<int>> ans;
};
```

> [!tip] `cur.resize(cur.size() - take)` 取代 `take` 次 `pop_back`
> 迴圈裡是**累加**取的個數（`take` 從 1 遞增到 `cnt`），`cur` 一路只進不出，所以撤銷放在迴圈外一次做完即可。若寫成每輪 `push` / `pop` 配對，就得重新在每輪塞回前面所有的 `v`，反而更繞。

> [!warning] 去重必須發生在搜尋當下，不能事後補
> 常見的偷懶版是不去重、最後把 `ans` 丟進 `set` 過濾。它會 AC，但重複的搜尋樹照樣整棵走完——量測顯示它多走了近 **1000 倍**的節點。`set` 只擦乾淨結果，省不到任何搜尋成本。

> [!note] 實測：呼叫次數少 ≠ 比較快
> 量測條件：`g++ -std=c++20 -O2`，各跑 20 次取平均，另以 5000 組隨機測資對「暴力枚舉所有子集再去重」交叉驗證，兩個方法輸出全數一致。
>
> **小規模**：`candidates` = 1…8 各 5 個（共 40 個），`target = 20`，296 組解
>
> | 版本 | 遞迴呼叫次數 | 時間 |
> | --- | --- | --- |
> | 方法一 同層跳過 | **1,569** | 0.030 ms |
> | 方法二 (值, 次數) 分組 | 2,682 | **0.027 ms** |
> | 不去重 + `set` 過濾 | 1,483,177 | 35.793 ms |
>
> **題目上限**：`candidates` = 1…10 各 10 個（共 100 個），`target = 30`，3145 組解
>
> | 版本 | 遞迴呼叫次數 | 時間 |
> | --- | --- | --- |
> | 方法一 同層跳過 | **18,578** | 0.426 ms |
> | 方法二 (值, 次數) 分組 | 32,187 | **0.259 ms** |
>
> 反直覺的結果：**節點數最少的方法一反而最慢**。方法一每層 for 迴圈要對整段重複值逐一 `continue` 空轉（10 個 `2` 就掃 10 次），方法二一次跳過整組；節點數的優勢被逐元素掃描吃掉。實務上兩者都在 1 ms 內、差異無意義，但「呼叫次數少不等於比較快」這件事本身值得記住。

## Related Problems

- [[0039-Combination-Sum]] — 同一個回溯骨架，候選相異且可重複取，`start` 單調遞增就夠去重
- [[0090-Subsets-II]] — 一模一樣的 `i > start` 同層跳過，只是把「湊出 target」換成「列出所有子集」
- [[0047-Permutations-II]] — 同樣是重複值去重，但排列題沒有 `start`，同層跳過得搭配 `used` 陣列
- [[0078-Subsets]] — 選／不選骨架的原型，無重複值、不必去重
- [[0015-3Sum]] — 排序後同層跳過相同值的雙指標版本，去重邏輯同源
