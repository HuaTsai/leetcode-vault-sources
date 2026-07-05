---
leetcode-id: 739
difficulty: medium
tags:
  - stack
  - array
  - monotonic-stack
  - grind-169
  - neetcode-150
memo: 「找右邊第一個更大」用單調遞減 stack；存 index 不存溫度（index 能還原值又能算天數差），且嚴格小於才彈出——等溫不算更暖
dg-publish: true
---

## Problem Description

Given an array of integers `temperatures` represents the daily temperatures, return an array `answer` such that `answer[i]` is the number of days you have to wait after the `ith` day to get a warmer temperature. If there is no future day for which this is possible, keep `answer[i] == 0` instead.

## Solution

核心觀念：題目翻譯過來就是「對每一天，找**右邊第一個比它大**的元素」——經典的 next greater element 模式，標準解法是**單調遞減 stack**。關鍵是視角反轉：不是「每天各自往右找答案」（O(n²) 的浪費在於好幾天都在等同一個暖天，卻各掃各的），而是「**新的一天進場，來結算舊帳**」。stack 存「還在等更暖天」的 index，每掃到新的一天，就把棧頂所有比它低溫的日子彈出結算；結算的天數差 `i - j` 正是答案。由於任何比棧頂暖的日子一進場就會把棧頂清掉，棧內溫度由底到頂**必然非嚴格遞減**，所以彈出時結算的一定是「右邊第一個更大」，不會跳過誰。

```txt
temps:  [73, 74, 75, 71, 69, 72, 76]
掃到 72（i=5）：69（i=4）< 72 → 彈出，ans[4] = 5-4 = 1
              71（i=3）< 72 → 彈出，ans[3] = 5-3 = 2
              75（i=2）≥ 72 → 停，5 入棧
棧（存 index，溫度由底到頂遞減）：[2, 5]，即溫度 75、72
```

> [!important] 雙重迴圈為什麼是 O(n)
> while 套在 for 裡長得像 O(n²)，但複雜度要數「元素進出棧的總次數」而不是迴圈層數：每個 index 恰好入棧一次、出棧至多一次，所有 while 迭代加總 ≤ n，攤銷下來整體 O(n)。這是單調 stack 類題共通的分析方式。

### 方法一：單調遞減 stack 存 index — O(n)／O(n)（推薦）

棧裡只存 index。判斷訊號：題目出現「下一個更大／第一個超過／要等幾天」這類措辭，就是這個模板。

```cpp
// Time: O(n)  每個 index 恰好入棧一次、出棧至多一次
// Space: O(n) 最壞溫度單調遞減，全部堆在棧裡
class Solution {
 public:
  vector<int> dailyTemperatures(vector<int>& temperatures) {
    int n = temperatures.size();
    vector<int> ans(n);
    stack<int> waiting;  // 還在等更暖天的 index，溫度由底到頂非嚴格遞減
    for (int i = 0; i < n; ++i) {
      while (!waiting.empty() && temperatures[waiting.top()] < temperatures[i]) {
        int j = waiting.top();
        waiting.pop();
        ans[j] = i - j;
      }
      waiting.push(i);
    }
    return ans;
  }
};
```

> [!tip] 存 index、不存值
> index 能還原溫度（`temperatures[j]`）、又能算天數差 `i - j`；溫度卻不能還原 index。index 是資訊量嚴格更大的那個，單調 stack 類題一律優先存 index，遇到要回頭取值、算距離的變體都直接適用。

> [!warning] 嚴格 `<` 才彈出——等溫不算更暖
> 相同溫度的日子要留在棧裡繼續等，所以棧內是「非嚴格」遞減。`[30,30,41]` 的答案是 `[2,1,0]`，兩個 30 都由 41 結算；手滑寫成 `<=` 就會讓第一個 30 被第二個 30 錯誤結算成 `[1,1,0]`。這是本題僅次於彈出順序的易錯點。

### 方法二：棧元素帶著溫度一起存（pair 版）— O(n)／O(n)

思路完全相同，差別只在棧存 `{溫度, index}`。條件式 `waiting.top().first < temperatures[i]` 少一層陣列索引、讀起來直接一點，但溫度是可由 index 推導的冗餘資料，每格棧元素多佔一倍空間。當「值」無法由 index 還原時（例如串流輸入、或原陣列之後會被修改），這種寫法才成為必要。

```cpp
// Time: O(n)
// Space: O(n)  每格存 pair，佔用是方法一的兩倍
class Solution {
 public:
  vector<int> dailyTemperatures(vector<int>& temperatures) {
    int n = temperatures.size();
    vector<int> ans(n);
    stack<pair<int, int>> waiting;  // {temperature, index}
    for (int i = 0; i < n; ++i) {
      while (!waiting.empty() && waiting.top().first < temperatures[i]) {
        int j = waiting.top().second;
        ans[j] = i - j;
        waiting.pop();
      }
      waiting.push({temperatures[i], i});
    }
    return ans;
  }
};
```

> [!note] 還有一個 O(1) 額外空間的變體
> 從右往左掃，利用已填好的 `ans` 陣列「跳躍」：檢查 `i` 右邊時，若 `temperatures[k]` 不夠暖且 `ans[k] > 0`，直接跳到 `k + ans[k]` 而不必逐格走。不需要 stack（不計輸出陣列則額外空間 O(1)），是面試加分項，但主線仍是單調 stack。

## Related Problems

[[0496-Next-Greater-Element-I]] — next greater element 的原型題，同款單調遞減 stack
[[0503-Next-Greater-Element-II]] — 環狀陣列變體，繞兩圈掃、index 取模
[[0901-Online-Stock-Span]] — 鏡像題：往左找連續「不高於」的跨度，單調 stack 的串流版
[[0084-Largest-Rectangle-in-Histogram]] — 單調 stack 集大成：對每根柱子找左右兩側第一個更矮的
