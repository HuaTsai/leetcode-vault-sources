---
leetcode-id: 84
difficulty: hard
tags:
  - monotonic-stack
  - range-maximum
  - maximum-query
  - array
  - grind-169
  - neetcode-150
memo: 每根柱子的面積只在它被彈出的那一刻結算一次（右界＝當前 i、左界＝彈出後的新棧頂），這是把 O(n²) 壓成 O(n) 的關鍵；尾端補一根高度 0 的哨兵才會 flush 棧內殘留的遞增柱子
dg-publish: true
---

## Problem Description

Given an array of integers `heights` representing the histogram's bar height where the width of each bar is `1`, return the area of the largest rectangle in the histogram.

## Solution

核心觀念：最大矩形的高度一定等於**某根柱子的高度**（不然還能往上長），所以答案 = 對每根柱子 `i` 問「以 `heights[i]` 為高，最寬能延伸到哪」。這個寬度由兩側**第一個比它矮**的柱子夾出來：左界 `L` = 左邊第一個更矮的 index、右界 `R` = 右邊第一個更矮的 index，寬度就是 `R - L - 1`。這正是 next smaller element 模式，用**單調遞增 stack** 一次掃完。

真正的關鍵不在算式，而在**結算時機**：一根柱子的右界，恰好在它「被彈出」的那一刻才確定——因為能把它彈出的 `heights[i]` 就是右邊第一個更矮的。所以每根柱子只在被 pop 時結算一次，總共 n 次結算，而不是每步重掃整個棧。

```txt
heights: [2, 1, 5, 6, 2, 3]
index:    0  1  2  3  4  5

掃到 2（i=4）：棧內 index [1, 2, 3]，高度 1 < 5 < 6
  6（i=3）> 2 → 彈出，彈完新棧頂是 2 → 左界 2、右界 4，寬 4-2-1 = 1，面積 6
  5（i=2）> 2 → 彈出，彈完新棧頂是 1 → 左界 1、右界 4，寬 4-1-1 = 2，面積 10  ← 答案
  1（i=1）≤ 2 → 停，4 入棧
```

> [!important] 「彈出後的新棧頂」就是左界
> 這是本題最漂亮的一步：棧內高度嚴格遞增，所以某個元素被彈掉後，露出來的新棧頂必然是它左邊第一個更矮的柱子——左界不必另外求，pop 這個動作自己就把它交出來了。棧空則左界視為 `-1`（左邊整片都比它高）。

### 方法一：單調遞增 stack 存 index + 尾端哨兵 — O(n)／O(n)（推薦）

```cpp
// Time: O(n)  每個 index 恰好入棧一次、出棧至多一次，攤銷 O(n)
// Space: O(n) 最壞情況（高度遞增）整個陣列都堆在棧裡
class Solution {
 public:
  int largestRectangleArea(vector<int>& heights) {
    int n = heights.size();
    int ans = 0;
    stack<int> st;  // 存 index，對應高度由底到頂遞增
    for (int i = 0; i <= n; ++i) {
      int cur = (i == n) ? 0 : heights[i];  // 尾端哨兵，逼出棧內殘留
      while (!st.empty() && heights[st.top()] >= cur) {
        int h = heights[st.top()];
        st.pop();
        int left = st.empty() ? -1 : st.top();  // 左邊第一個更矮的 index
        ans = max(ans, h * (i - left - 1));
      }
      st.push(i);
    }
    return ans;
  }
};
```

> [!warning] 沒有哨兵，遞增的輸入會回傳 0
> `[1,2,3]` 這種全程不觸發 pop 的輸入，迴圈跑完後三根柱子還躺在棧裡，一次都沒結算過。讓 `i` 多跑一輪到 `n`、把高度視為 `0`，就能逼所有殘留元素彈出結算，比在迴圈外複製一份 flush 程式碼乾淨得多。

> [!tip] 彈出條件用 `>=` 或 `>` 都對
> 等高時提早彈出的那根，寬度會算少（左界被同高的鄰居擋住）；但等高群組中**最右邊那根**保證能延伸過所有同高鄰居，拿到完整寬度，最大值不受影響。`[2,2,2]` 兩種寫法都得 6。這點和 [[0739-Daily-Temperatures]] 不同——那題每個 index 都要有自己正確的答案，等溫誤彈就直接錯；本題只取全域最大值，才容得下這個鬆散。

### 方法二：索引與高度分存兩個 vector — O(n)／O(n)

同樣的演算法，只是把「高度」另外存一份，彈出時不必回頭查 `heights`。實測速度和方法一改用 `vector` 當棧的版本相同（見下方 tip），所以這不是效能取捨，而是適用範圍：當原陣列之後會被修改、或高度是串流進來無法回查時，這種寫法才成為必要，代價是兩個容器要同步 push／pop，多一份記憶體也多一個出錯點。`indices` 底部放 `-1` 當哨兵，維持不變量：`values[j]` 的左界恰好是 `indices[j]`、自己的位置是 `indices[j + 1]`。彈出時先 `indices.pop_back()` 彈掉自己的位置，露出來的 `indices.back()` 就是左界。

```cpp
// Time: O(n)
// Space: O(n)  兩個 vector，常數比方法一大一倍
class Solution {
 public:
  int largestRectangleArea(vector<int>& heights) {
    vector<int> indices = {-1};  // 底部哨兵，代表「左邊沒有更矮的柱子」
    vector<int> values;          // values[j] 的左界恰好是 indices[j]
    int n = heights.size();
    int ans = 0;
    for (int i = 0; i <= n; ++i) {
      int cur = (i == n) ? 0 : heights[i];
      while (!values.empty() && values.back() >= cur) {
        int h = values.back();
        values.pop_back();
        indices.pop_back();  // 彈掉自己的 index 後，indices.back() 就是左界
        ans = max(ans, h * (i - 1 - indices.back()));
      }
      indices.push_back(i);
      values.push_back(cur);
    }
    return ans;
  }
};
```

> [!warning] 把結算放在 while 外面就會 TLE
> 常見的錯法是「每讀一根柱子，就重掃整個棧算一次面積」：
>
> ```cpp
> indices.push_back(i);
> values.push_back(heights[i]);
> for (int j = 0; j < values.size(); ++j)  // ← 每個 i 都重掃整個棧
>   ans = max(ans, values[j] * (i - indices[j]));
> ```
>
> 這段算式本身是對的（`indices[j]` 確實是左界，且棧內元素都 ≤ `heights[i]`，右界能延伸到 `i`），錯的是重算了 n 次。輸入非遞減時（`[1,1,2,2,3,3,…]`）永遠不觸發 pop，棧長就是 `i`，退化成 O(n²)。實測 `n = 1e5`、高度非遞減：這版 2789 ms，方法一 0.2 ms。結算要搬進 while 迴圈裡，讓每根柱子只算一次。

> [!tip] `stack<int>` 的底層是 `deque`，不是 `vector`
> 單調棧只需要「尾端 push／pop／peek」，用 `vector<int>` 加 `back()`／`pop_back()`／`push_back()` 就夠，而且更快——`std::stack` 預設容器是 `std::deque`，多一層分段索引。同一台機器 `n = 1e5` 各跑 25 次取最佳：棧很深的非遞減輸入 0.260 ms（deque）對 0.116 ms（vector），進出頻繁的鋸齒輸入 0.186 ms 對 0.122 ms；隨機輸入則落在噪音內（0.766 對 0.741），因為時間都花在 cache miss 上。要保留 `std::stack` 的語意又要速度，寫 `stack<int, vector<int>>`。詳見 [[Micro-Optimization-Myths]]。

> [!note] 還有一個不用 stack 的寫法
> 先用兩趟掃描各自求出每根柱子的左界、右界陣列（求左界時，若 `heights[j] >= heights[i]` 就直接跳到 `left[j]`，跳過整段已知更高的區塊），再一趟算面積。同樣 O(n)、程式碼更好懂，但要三趟掃描與兩條額外陣列，主線仍是單調 stack。

## Related Problems

[[0739-Daily-Temperatures]] — 同款單調 stack 的入門型：找右邊第一個更大，結算的是天數差而非面積
[[0042-Trapping-Rain-Water]] — 單調遞減 stack 的鏡像題，彈出時結算的是凹槽而非矩形
[[0085-Maximal-Rectangle]] — 把矩陣每一列壓成直方圖高度，逐列直接套用本題，是最直接的升級版
[[0853-Car-Fleet]] — 同為「新元素進場結算舊帳」的視角，但改成由右往左掃
[[1793-Maximum-Score-of-a-Good-Subarray]] — 「以某元素為區間最小值能延伸多寬」的同構問題，多一個必須包含 `k` 的限制
