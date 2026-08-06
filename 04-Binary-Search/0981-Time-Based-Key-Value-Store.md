---
leetcode-id: 981
difficulty: medium
tags:
  - binary-search
  - hash
  - string
  - design
  - grind-169
  - neetcode-150
memo: 題目保證每個 key 的 timestamp 嚴格遞增，所以 push_back 天然有序，查詢就是「找最後一個不大於 target」＝ upper_bound 再退一格；易錯在 comp 的參數順序，lower_bound 是（元素，值）而 upper_bound 是（值，元素），且 ranges 版拒收異質 comp、要改用 proj
dg-publish: true
---

## Problem Description

Design a time-based key-value data structure that can store multiple values for the same key at different time stamps and retrieve the key's value at a certain timestamp.

Implement the `TimeMap` class:

- `TimeMap()` Initializes the object of the data structure.
- `void set(String key, String value, int timestamp)` Stores the key `key` with the value `value` at the given time `timestamp`.
- `String get(String key, int timestamp)` Returns a value such that `set` was called previously, with `timestamp_prev <= timestamp`. If there are multiple such values, it returns the value associated with the largest `timestamp_prev`. If there are no values, it returns `""`.

Constraints:

- key and value consist of lowercase English letters and digits.
- All the timestamps timestamp of set are strictly increasing.

## Solution

核心觀念：constraints 那句 **「All the timestamps timestamp of set are strictly increasing」是整題的關鍵禮物**——它保證同一個 key 的 timestamp 是按呼叫順序遞增的，所以 `push_back` 進 vector 就天然有序，完全不需要排序。剩下的查詢語意是「**找最後一個 `timestamp ≤ target`**」，而這正是 `upper_bound` 退一格。

```txt
key = "foo"
  timestamp    1       4       9
  value       bar     bar2    bar3

  get("foo", 7)：找最後一個 timestamp ≤ 7
    upper_bound(7) ──→ 指向 9（第一個 > 7）
    prev 退一格    ──→ 指向 4  → 回傳 "bar2"

  get("foo", 0)：upper_bound(0) 指向 1，剛好 == begin()
                 代表沒有任何 timestamp ≤ 0 → 回傳 ""
```

> [!important] 「找最後一個 `≤ x`」不要自己手寫二分
> 標準寫法永遠是 **`upper_bound(x)` 再 `prev`**。`upper_bound` 回傳第一個 `> x`，往前退一格自然就是最後一個 `≤ x`。唯一要處理的邊界是 `it == begin()`，代表整段都 `> x`、答案不存在。
>
> 對照 [[Binary-Search-Templates]]：手寫版屬於模板二的「找最後一個滿足條件」，要用 `l = m` 配上取整；而借 `upper_bound` 就把那組容易配錯的規則整包外包出去了。**能用 STL 就別手寫**，這題的難度應該花在容器設計上，不是二分邊界上。

### 方法一：vector + STL 二分 — set O(1)／get O(log n)（推薦）

```cpp
// Time: set O(1) 均攤，get O(log n)
// Space: O(n)
class TimeMap {
 public:
  void set(string key, string value, int timestamp) {
    data_[key].push_back({move(value), timestamp});
  }

  string get(const string& key, int timestamp) const {
    if (!data_.contains(key)) return "";
    const auto& vec = data_.at(key);
    auto it = ranges::upper_bound(vec, timestamp, {}, &pair<string, int>::second);
    if (it == vec.begin()) return "";
    return prev(it)->first;
  }

 private:
  unordered_map<string, vector<pair<string, int>>> data_;  // key -> (value, timestamp)
};
```

> [!note] 為什麼是 `data_.at(key)` 而不是 `data_[key]`
> `contains` + `at` 確實查了兩次雜湊表（`find` 一次就夠），但換來可讀性，這個取捨合理。真正該避免的是 `operator[]`：它是**非 const** 的（找不到就插入），一旦用了，`get` 就沒辦法宣告成 `const`。`at` 有 const 多載，語意也更誠實——「我確定它在，不在就是邏輯錯誤」。

### 方法二：`map<int, string>` — set O(log n)／get O(log n)

把每個 key 的歷史直接存成 `map<int, string>`，時間戳就是 map 的 key。這樣**根本不需要 comparator 或 projection**，因為要搜的東西本身就是容器的 key：

```cpp
// Time: set O(log n)，get O(log n)
// Space: O(n)
class TimeMap {
 public:
  void set(string key, string value, int timestamp) {
    data_[key][timestamp] = move(value);
  }

  string get(const string& key, int timestamp) const {
    if (!data_.contains(key)) return "";
    const auto& hist = data_.at(key);
    auto it = hist.upper_bound(timestamp);
    if (it == hist.begin()) return "";
    return prev(it)->second;
  }

 private:
  unordered_map<string, map<int, string>> data_;  // key -> (timestamp -> value)
};
```

> [!tip] comparator 的複雜度往往是「容器選錯」的症狀
> 方法一之所以要寫 projection，是因為搜尋鍵 `timestamp` 被埋在 `pair` 的第二個欄位裡，得先把它挖出來。方法二讓搜尋鍵**就是容器的 key**，適配層整個消失。下次發現自己在為了 `lower_bound` 寫奇怪的 comparator 時，先回頭問一句：是不是資料擺錯位置了？

兩者的取捨很清楚：

|          | 方法一 vector                  | 方法二 map                 |
| -------- | ------------------------------ | -------------------------- |
| `set`    | O(1) 均攤                      | O(log n)                   |
| `get`    | O(log n)                       | O(log n)                   |
| 記憶體   | 連續，cache 友善               | 每筆一個節點，額外指標開銷 |
| 程式碼   | 要處理 pair + projection       | 沒有適配層，最短           |
| 用到保證 | **有**（靠遞增才能 push_back） | 沒有（亂序插入也對）       |

> [!warning] 方法二等於浪費掉題目給的保證
> 「timestamps strictly increasing」這條 constraint 對方法二毫無作用——就算時間戳亂序丟進來 map 照樣正確。**題目特地寫這句，就是在暗示 vector + push_back 這條路。**面試時兩種都能過，但方法一才是它想考的；能講出「因為題目保證遞增，所以我不需要排序也不需要 map」是加分的。

### 附錄：`lower_bound` / `upper_bound` 的 comp 與 proj

這題最容易踩的其實不是演算法，是 STL 介面。

**語意**（前提：區間已依同一個序排好）

| 寫法                      | 得到                      |
| ------------------------- | ------------------------- |
| `lower_bound(v, x)`       | 第一個 **`≥ x`**          |
| `upper_bound(v, x)`       | 第一個 **`> x`**          |
| `prev(lower_bound(v, x))` | 最後一個 **`< x`**        |
| `prev(upper_bound(v, x))` | 最後一個 **`≤ x`** ← 本題 |

> [!warning] 陷阱一：兩者 comparator 的參數順序**相反**
> 掛 log 印出實際呼叫：
>
> ```txt
> std::lower_bound(v, 4)  ->  comp(元素 b, 值 4)     元素在前
> std::upper_bound(v, 4)  ->  comp(值 4, 元素 b)     值在前
> ```
>
> 記法：**comp 的參數順序永遠對應「序列上誰排在前面」。**
>
> - `lower_bound` 問的是「元素是否排在 x 前面」→ `comp(元素, x)`
> - `upper_bound` 問的是「x 是否排在元素前面」→ `comp(x, 元素)`
>
> 所以只有 `upper_bound` 的 comparator 第一個參數是**你要找的值**。寫成 `[](const auto& a, const auto& b){ return a.second < b.second; }` 會得到 `error: request for member 'second' in 'a', which is of non-class type 'const int'`——因為 `a` 根本是那個 `int`。

> [!warning] 陷阱二：`std::` 收異質 comparator，`std::ranges::` 不收
> `std::upper_bound` 的 comparator 兩個參數型別可以不同（`int` 和 `pair`），它只是被呼叫而已。
>
> 但 `ranges::upper_bound` 的 comp 受 concept `indirect_strict_weak_order` 約束，**必須是同一型別上的嚴格弱序**——編譯器會檢查 `comp(a, a)`、`comp(b, b)` 也合法。所以就算把參數順序改對成 `[](int t, const auto& p){ return t < p.second; }`，`ranges` 版**照樣編不過**：
>
> ```txt
> error: request for member 'second' in 'p', which is of non-class type 'const int'
> ```
>
> 異質比較在 `ranges` 的正解不是 comparator，是 **projection**。

**projection 是什麼**

`ranges` 演算法的最後一個參數，作用是「**比較之前先把元素變換一次**」。comp 留 `{}` 表示沿用預設的 `ranges::less`：

```cpp
ranges::upper_bound(vec, timestamp, {}, &pair<string, int>::second);
//                                  ↑        ↑
//                          comp 用預設   proj 指定「拿 .second 去比」
```

掛 log 看實際順序：

```txt
ranges::upper_bound(v, 4, comp, proj)
    proj(b) -> 4
  comp(值 4, proj(元素) 4)
    proj(c) -> 9
  comp(值 4, proj(元素) 9)
```

> [!important] proj 只套在**元素**上，不套在你要找的值上
> 所以 `ranges::upper_bound(vec, timestamp, {}, &pair<string,int>::second)` 實際比的是 `timestamp < p.second`——`int` 對 `int`，型別自然對齊。**這就是為什麼異質比較該用 proj 而不是 comp**：proj 把元素投影成和 value 同型，同質性就自動成立了。

proj 可以是成員指標、成員函式指標，或任何一元可呼叫物：

```cpp
ranges::sort(people, {}, &Person::age);                     // 依 age 排序
ranges::max_element(v, {}, &pair<string, int>::second);     // 依 second 找最大
ranges::find(people, "bob", &Person::name);                 // 依 name 找
ranges::sort(words, greater{}, [](const string& s) { return s.size(); });  // comp + proj 併用
```

**`std::` 與 `std::ranges::` 對照**

|              | `std::`                   | `std::ranges::`                       |
| ------------ | ------------------------- | ------------------------------------- |
| 傳入         | `first, last`             | 整個容器（也接受 `first, last`）      |
| 異質 comp    | 可以                      | **編不過**，改用 proj                 |
| 取欄位比較   | 只能自己寫 comparator     | `{}` + proj，更短也更難寫錯           |
| 傳入右值容器 | 照常回 iterator，可能懸空 | 回 `ranges::dangling`，**編譯期擋下** |

> [!warning] 陷阱三：方法二用了 `map`，千萬別對它套泛型版
> `std::map` 自帶成員 `lower_bound` / `upper_bound`，走紅黑樹是 O(log n)。而 `std::lower_bound(m.begin(), m.end(), x, comp)` **也能編譯通過**，但 map 的迭代器只有 bidirectional，二分需要的跳躍退化成一步一步走，整體變 O(n)——比較次數仍是 log n，被吃掉的是迭代器移動。
>
> 完整的機制、實測數字，以及同一家族的其他陷阱（`std::find` 慢 1600 倍、comparator 非嚴格弱序會段錯誤、`remove` 不會刪除元素等）見 [[STL-Pitfalls]]。

## Related Problems

[[Binary-Search-Templates]] — 本題的查詢語意是「找最後一個 `≤ target`」，屬模板二，但外包給 `upper_bound` + `prev`；手寫版的取整配對規則見該篇
[[0034-Find-First-and-Last-Position-of-Element-in-Sorted-Array]] — 手寫 `lower_bound` / `upper_bound` 的練習題，正好就是這對函式的定義本身
[[0035-Search-Insert-Position]] — 就是 `lower_bound` 本人，理解「第一個 `≥ x`」語意的最短路徑
[[0704-Binary-Search]] — 模板一原型，對照「有明確出口」與本題「找邊界」的差別
[[0146-LRU-Cache]] — 同為 design 題，難度一樣不在演算法而在容器組合的選擇
[[STL-Pitfalls]] — 本篇附錄講的是 comp 與 proj 的**用法**；那篇收的是「編得過但複雜度或語意錯」的各類陷阱
