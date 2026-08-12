---
leetcode-id: 853
difficulty: medium
tags:
  - stack
  - monotonic-stack
  - array
  - sort
  - neetcode-150
memo: 把每台車換算成到 target 的時間，依起點由前往後掃，後車時間嚴格大於前方車隊才自成新車隊，否則併入（相等視為恰好追上）。
dg-publish: true
---

## Problem Description

There are `n` cars at given miles away from the starting mile 0, traveling to reach the mile `target`.

You are given two integer arrays `position` and `speed`, both of length `n`, where `position[i]` is the starting mile of the `ith` car and `speed[i]` is the speed of the `ith` car in miles per hour.

A car cannot pass another car, but it can catch up and then travel next to it at the speed of the slower car.

A car fleet is a single car or a group of cars driving next to each other. The speed of the car fleet is the minimum speed of any car in the fleet.

If a car catches up to a car fleet at the mile `target`, it will still be considered as part of the car fleet.

Return the number of car fleets that will arrive at the destination.

Constraints: All the values of `position` are unique.

## Solution

核心觀念：車不能超車，後車追上前車後就併成同一車隊、以較慢的前車速度前進。與其模擬追撞，不如把每台車換算成**到達 `target` 所需的時間** `t = (target - position) / speed`，再依**起點位置由前往後（position 由大到小）**掃描。

由前往後看，若某台車的到達時間**大於**目前前方車隊的到達時間，代表它比前車慢、永遠追不上 → 自成一個新車隊；若**小於等於**，代表它會在 `target` 之前（或恰好在 `target`）追上 → 併入前方車隊。前方車隊的到達時間 = 車隊中最慢（最大時間）者，正是掃描時遇到的第一台（最前面）車。

```txt
target = 12
起點:  0    3    5    8   10   → 往右到 target
由最靠近 target 的車往回掃，front = 前方車隊的到達時間：

 p=10  t=(12-10)/2 = 1.0   > front(−∞) → 新車隊 #1，front=1.0
 p= 8  t=(12- 8)/4 = 1.0   ≤ 1.0       → 併入 #1（恰好同時到達）
 p= 5  t=(12- 5)/1 = 7.0   > 1.0       → 新車隊 #2，front=7.0
 p= 3  t=(12- 3)/3 = 3.0   ≤ 7.0       → 併入 #2（3<7 追得上）
 p= 0  t=(12- 0)/1 =12.0   > 7.0       → 新車隊 #3，front=12.0
答案 = 3
```

> [!important]
> 化「位置 + 速度」為「到達時間」是本題關鍵：一旦排序，車隊數就等於掃描過程中「到達時間創新高」的次數。

> [!tip]
> 用**嚴格 `>`** 判斷新車隊——時間相等表示恰好在 `target` 追上，題目視為同一車隊，必須併入而非新開。

### 方法一：Monotonic Stack — O(n log n)／O(n)

最正規的寫法：把到達時間壓進 stack，stack 由底到頂遞增，每個元素代表一個獨立車隊；後車時間 `> top` 才是新車隊，否則被前方車隊吸收。最後 stack 大小就是答案。

```cpp
// Time: O(nlogn)
// Space: O(n)
class Solution {
 public:
  int carFleet(int target, vector<int>& position, vector<int>& speed) {
    int n = position.size();
    vector<pair<int, int>> cars(n);
    for (int i = 0; i < n; ++i) cars[i] = {position[i], speed[i]};
    ranges::sort(cars, ranges::greater{});  // 依起點位置由大到小（最靠近 target 的在前）
    stack<double> st;
    for (auto& [pos, spd] : cars) {
      double time = static_cast<double>(target - pos) / spd;
      if (st.empty() || time > st.top()) st.push(time);  // 追不上前方車隊 → 自成新車隊
      // 否則會在 target 前（或恰好）追上 → 併入前方車隊，不入堆疊
    }
    return st.size();
  }
};
```

### 方法二：追蹤最大到達時間（不用堆疊）— O(n log n)／O(n)

其實不必真的維護一個 stack：只要一路記著「前方車隊的到達時間」`tlimit`，遇到更大的就 `++ans` 並更新。這是使用者原本的解法，用 `views::zip` 把 `position` 與 `times` 綁在一起排序，只讀排序後的 `times`。

```cpp
// Time: O(nlogn)
// Space: O(n)
class Solution {
 public:
  int carFleet(int target, vector<int>& position, vector<int>& speed) {
    int n = position.size();
    vector<double> times(n);
    for (int i = 0; i < n; ++i) {
      times[i] = static_cast<double>(target - position[i]) / speed[i];
    }
    ranges::sort(views::zip(position, times), ranges::greater{}, [](auto &&e) { return std::get<0>(e); });
    int ans = 0;
    double tlimit = numeric_limits<double>::lowest();
    for (const auto &t : times) {
      if (t > tlimit) {
        ++ans;
        tlimit = t;
      }
    }
    return ans;
  }
};
```

> [!warning]
> 兩個容易踩雷的點：
>
> 1. 初始下界用 `lowest()` 不要用 `min()`：對浮點數 `min()` 是「最小**正**規格化值」（≈ 2.2e-308）而非最負值，`lowest()` 才是最負有限值（`= -max()`）。這題所有到達時間都 > 0，用 `min()` 也碰巧會過，但語意上錯的。
> 2. `views::zip` 是 **C++23**，用 `-std=c++20` 會編不過。若要 C++20，改成排序 `vector<pair>`（如方法一）即可。

### 方法三：Counting Sort — O(n + target)／O(target)

`n log n` 全來自「依位置排序」；只要注意 `position` 是 `[0, target)` 內的**相異整數**，就能用桶排取代比較排序，把 log 拿掉。把每台車速丟進以位置為索引的桶，再由靠近 `target` 往回掃即可。

把每台車看成 2D 點 `(position, time)`，車隊領頭車就是「右上方沒有人壓過它」的**極大點（Pareto front）**；桶排讓我們免排序、直接按位置遞減走一遍座標階梯。

```cpp
// Time: O(n + target)
// Space: O(target)
class Solution {
 public:
  int carFleet(int target, vector<int>& position, vector<int>& speed) {
    vector<int> spd(target, -1);              // spd[p] = 位置 p 的車速，-1 表示沒車
    for (int i = 0; i < static_cast<int>(position.size()); ++i) {
      spd[position[i]] = speed[i];
    }
    int ans = 0;
    double tlimit = -1.0;
    for (int p = target - 1; p >= 0; --p) {   // 由最靠近 target 往回掃
      if (spd[p] == -1) continue;
      double time = static_cast<double>(target - p) / spd[p];
      if (time > tlimit) {
        ++ans;
        tlimit = time;
      }
    }
    return ans;
  }
};
```

> [!note]
> 這是「位置密集」時的特化，不是無腦更快。當 `target ≫ n`（如 `n=10`、`target=10^6`）時，`O(target)` 的時間與記憶體反而比方法一的排序版更差。只有 `target = O(n)` 時才是真正划算的線性解，實務預設仍選方法一。

> [!tip]
> 純比較模型下本題沒有 o(n log n) 解——找領頭車等價於算 2D 極大點集，在代數決策樹模型有 Ω(n log n) 下界。方法三能變線性，全靠「整數且有界」這個額外前提繞過比較排序。

## Related Problems

- [[1776-Car-Fleet-II]] — 直接的進階版 follow-up，改求「每台車追上前車的時刻」，需用單調堆疊維護碰撞時間。
- [[0739-Daily-Temperatures]] — 同樣用 stack 維護「尚未被解決」的元素，比較後決定是否彈出／合併。
- [[0084-Largest-Rectangle-in-Histogram]] — monotonic stack 經典題，練習用堆疊維護遞增／遞減序列。
