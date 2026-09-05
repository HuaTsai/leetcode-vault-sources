---
leetcode-id: 90
difficulty: medium
tags:
  - array
  - backtracking
  - bit-manipulation
  - neetcode-150
memo: 排序讓重複值相鄰，同一層 for 迴圈裡一串相同值只讓第一個當代表；去重條件是 `i > start` 而非 `i > 0`，因為 `i == start` 時的 `nums[i - 1]` 是上層選走、還留在 `cur` 裡的，跳掉就取不出 `[2,2]` 這種同值取兩個的子集
dg-publish: true
---

## Problem Description

Given an integer array `nums` that may contain duplicates, return all possible subsets (the power set).

The solution set must not contain duplicate subsets. Return the solution in any order.

## Solution

核心觀念：骨架就是 [[0078-Subsets]]，唯一新增的是**重複值去重**。排序讓相同值相鄰後，規則和 [[0040-Combination-Sum-II]] 一字不差——**同一層裡，一串相同的值只讓第一個當代表**。理由是同層兩個相同值往 `cur` 放的東西一樣，差別只在往後能搜的範圍，後者的範圍是前者的真子集，只會生出重複。

```txt
nums = [1, 2, 2]（已排序）    每個節點的 cur 都是一組答案

bt(start=0) cur=[]                                        ← 收錄 []
├─ i=0 取 1 ── bt(1) cur=[1]                              ← 收錄 [1]
│               ├─ i=1 取 2 ── bt(2) cur=[1,2]            ← 收錄 [1,2]
│               │               └─ i=2 取 2 ── bt(3)      ← 收錄 [1,2,2]
│               └─ i=2 SKIP（i>start=1，nums[2]==nums[1]）
├─ i=1 取 2 ── bt(2) cur=[2]                              ← 收錄 [2]
│               └─ i=2 取 2 ── bt(3) cur=[2,2]            ← 收錄 [2,2]
└─ i=2 SKIP（i>start=0，nums[2]==nums[1]）
```

> [!important] 「同層」是兄弟，不是祖先
> `[1,2,2]` 和 `[2,2]` 這種「同一個值取兩個」的子集**照樣會被產出來**，因為第二個 `2` 是從**上一層取完往下傳**的，在 `bt(2)` 那層的迴圈裡 `i == start`，`i > start` 為 false，比較根本不會執行。被跳掉的只有「同一個 for 迴圈裡，第二個以後的相同值」——它的子樹跟第一個那條完全重疊。

以下兩個方法複雜度相同：`n` 個元素最多 `2^n` 個子集，每收錄一組要複製長度 `O(n)` 的 `cur`。

### 方法一：排序後回溯，同層跳過重複值 — O(n·2^n)／O(n)

子集樹的**每個節點都是答案**，所以進門就 `push`，不必走到底才收。

```cpp
// Time: O(n2^n)  2^n 個子集，每組複製 O(n)
// Space: O(n)    遞迴堆疊 + cur，不計輸出
class Solution {
 public:
  vector<vector<int>> subsetsWithDup(vector<int>& numbers) {
    nums = numbers;
    ranges::sort(nums);  // 讓重複值相鄰
    bt(0);
    return ans;
  }

 private:
  void bt(int start) {
    ans.push_back(cur);  // 每個節點都是一組答案，不需要 base case
    for (int i = start; i < (int)nums.size(); ++i) {
      if (i > start && nums[i] == nums[i - 1]) continue;  // 同層只讓第一個代表
      cur.push_back(nums[i]);
      bt(i + 1);  // i + 1：每個下標只用一次
      cur.pop_back();
    }
  }

  vector<int> nums, cur;
  vector<vector<int>> ans;
};
```

> [!warning] 是 `i > start`，不是 `i > 0`
> 兩者語意天差地別：
>
> | 條件 | `nums[i - 1]` 的身分 | 該不該跳 |
> | --- | --- | --- |
> | `i > start` | 本層剛遞迴完又 `pop_back` 撤銷的兄弟，子樹已整棵走過 | 跳，否則整棵重複 |
> | `i == start` | 上一層選走的，**還留在 `cur` 裡** | 不能跳，這正是「同值取兩個」 |
>
> 實測 `nums = [1,2,2]`，正確答案 6 組；寫成 `i > 0` 只剩 `[] [1] [1,2] [2]` 四組——`[1,2,2]` 和 `[2,2]` 全滅，正好是所有「同值取兩個」的組合。

### 方法二：迭代，重複值只擴充上一輪新增的區段 — O(n·2^n)／O(1)

[[0078-Subsets]] 迭代法（每遇一數就複製現有全部子集再附加該數）的直接延伸。遇到跟前一個相同的值時，若還是複製**全部**既有子集，就會把「上上輪就存在、已經被前一個相同值擴充過」的那些再擴充一次，產生重複；只複製**上一輪新增的那一段**即可。

```txt
nums = [1, 2, 2]

i=0 值 1，非重複 → 擴充全部 [0,1)   ans = []  | [1]
                                          ↑ prev=1，上一輪新增區段 = [1,2)
i=1 值 2，非重複 → 擴充全部 [0,2)   ans = [] [1] | [2] [1,2]
                                                ↑ prev=2，新增區段 = [2,4)
i=2 值 2，與前一個相同 → 只擴充 [2,4) ans = [] [1] [2] [1,2] | [2,2] [1,2,2]
                        若擴充 [0,4) 會再生出一次 [2] 和 [1,2]
```

```cpp
// Time: O(n2^n)
// Space: O(1)  不計輸出
class Solution {
 public:
  vector<vector<int>> subsetsWithDup(vector<int>& numbers) {
    vector<int> nums = numbers;
    ranges::sort(nums);
    vector<vector<int>> ans{{}};
    int prev = 0;  // 上一輪新增子集的起始下標
    for (int i = 0; i < (int)nums.size(); ++i) {
      int sz = ans.size();
      int from = (i > 0 && nums[i] == nums[i - 1]) ? prev : 0;
      prev = sz;
      // C++17 後，emplace_back 回傳最後一項的 reference
      for (int k = from; k < sz; ++k) ans.emplace_back(ans[k]).emplace_back(nums[i]);
    }
    return ans;
  }
};
```

> [!tip] `prev` 就是方法一「同層只讓第一個代表」的迭代版
> 方法一靠 `i > start` 在**同一層**擋掉重複分支，這裡靠 `from = prev` 把擴充範圍限縮在**上一輪的新增段**，兩者砍掉的是同一批重複。差別只是遞迴由上往下展開、迭代由左往右堆疊。

> [!note] `ans.emplace_back(ans[k])` 的自我參照是安全的
> 參數 `ans[k]` 是容器自己的元素，但標準要求 `vector` 的 `push_back` / `emplace_back` 在需要重新配置時，先在新緩衝區建構好新元素才釋放舊的，所以不會讀到已失效的參考。

## Related Problems

- [[0078-Subsets]] — 同一題的無重複值版，本題只多了排序＋同層去重
- [[0040-Combination-Sum-II]] — 一模一樣的 `i > start` 同層跳過，只是把「列出所有子集」換成「湊出 target」
- [[0039-Combination-Sum]] — 同一個回溯骨架，候選相異可重複取，`start` 單調遞增就夠去重
- [[0047-Permutations-II]] — 同樣是重複值去重，但排列沒有 `start`，同層跳過得搭配 `used` 陣列
- [[0015-3Sum]] — 排序後同層跳過相同值的雙指標版本，去重邏輯同源
- [[Backtracking-Templates]] — 本題同時是「模板一 · 子集型」與「模板四 · 選／不選」的代表，兩套骨架的分界與混用反例都在這篇
