---
leetcode-id: 567
difficulty: medium
tags:
  - sliding-window
  - two-pointers
  - hash
  - string
  - neetcode-150
memo: 定長窗口的進出配對，右界推到 r 時進 s2[r]、出 s2[r−n1]（上一個窗的左界，不是新窗的左界）；把首窗放在迴圈外先比一次、主迴圈從 r＝n1 起跑，就不必寫「第一輪不滑動」的守衛，那個差一格的坑自然消失；array 可直接用 == 比整表，進階版維護 matches 把每步的 O(26) 降成 O(1)
dg-publish: true
---

## Problem Description

Given two strings `s1` and `s2`, return `true` if `s2` contains a permutation of `s1`, or `false` otherwise.

In other words, return `true` if one of `s1`'s permutations is the substring of `s2`.

Constraints:

- `s1` and `s2` consist of lowercase English letters.

## Solution

核心觀念：**排列只在乎每個字母出現幾次、不在乎順序**，所以「s1 的某個排列是 s2 的子字串」等價於——

```txt
s2 裡存在一段長度剛好 n1 的子字串，它的 26 格字母計數與 s1 完全相同
```

窗長固定成 `n1`，這讓本題比 [[0003-Longest-Substring-Without-Repeating-Characters]]、[[0424-Longest-Repeating-Character-Replacement]] 更單純：**沒有 `while` 收縮**。每前進一格就是「右邊進一個、左邊出一個」，窗長永遠不變。

```txt
s1 = "ab"（n1 = 2）
s2 =  e i d b a o o o
      0 1 2 3 4 5 6 7

 r=1  窗[0,1] = e i      首窗，先比一次
 r=2  進 s2[2]='d'，出 s2[0]='e'  →  窗[1,2] = i d
 r=3  進 s2[3]='b'，出 s2[1]='i'  →  窗[2,3] = d b
 r=4  進 s2[4]='a'，出 s2[2]='d'  →  窗[3,4] = b a   ← 計數 {a,b} 命中
```

> [!important] 定長窗的進出配對
> 窗口是 `[r - n1 + 1, r]`。右界推進到 `r` 時，**進來的是 `s2[r]`，出去的是 `s2[r - n1]`**——出去的那個是**上一個窗的左界**，不是新窗的左界。新窗的左界 `s2[r - n1 + 1]` 還好端端待在窗裡。

### 方法一：定長窗 + 整表比較 — O(26·n2)／O(1)（推薦）

把首窗放在迴圈外先比一次，主迴圈直接從 `r = n1` 起跑。這樣就不必在迴圈裡寫「第一輪不要滑動」的守衛，少一個分支、也少一個踩錯索引的機會。

```cpp
// Time: O(26 · n2)（每個位置比一次 26 格的計數表）
// Space: O(1)
class Solution {
 public:
  bool checkInclusion(string s1, string s2) {
    int n1 = s1.size(), n2 = s2.size();
    if (n1 > n2) return false;
    array<int, 26> need{}, win{};
    for (int i = 0; i < n1; ++i) {  // 首窗 [0, n1-1]
      ++need[s1[i] - 'a'];
      ++win[s2[i] - 'a'];
    }
    if (need == win) return true;
    for (int r = n1; r < n2; ++r) {  // 窗口 [r-n1+1, r]
      ++win[s2[r] - 'a'];            // 進：新右界
      --win[s2[r - n1] - 'a'];       // 出：舊左界
      if (need == win) return true;
    }
    return false;
  }
};
```

> [!tip] `array` 有 `operator==`，不要手寫 26 圈比較
> `if (need == win)` 完全等價於下面這段，成本同樣是 O(26)，但省掉一段容易寫錯的迴圈（忘了 `break`、忘了重設 `valid`）：
>
> ```cpp
> bool valid = true;
> for (int i = 0; i < 26; ++i) {
>   if (need[i] != win[i]) { valid = false; break; }
> }
> ```
>
> `array<int, 26>` 與 `vector<int>` 都支援 `==`，是逐元素比較。

> [!warning] 最常見的錯：出去的索引寫成**新窗**左界
>
> ```cpp
> ++win[s2[r] - 'a'];
> --win[s2[r - n1 + 1] - 'a'];  // ✗ 這是新窗的左界，還在窗裡
> --win[s2[r - n1] - 'a'];      // ✓ 舊窗的左界才是真正離開的那個
> ```
>
> 錯版等於「右邊加一個、順手把左邊第一個也扣掉」，窗口從此不對應任何真實子字串，計數甚至會變成負數。
>
> 用官方範例 `s1 = "ab"`、`s2 = "eidbaooo"` 走一步就歪了：首窗 `win = {e, i}`，`r=2` 時 `++d`、`--s2[1]='i'` 得到 `{e, d}`，但正確的窗 `"id"` 應該是 `{i, d}`。**這個 bug 連 LeetCode 第一個範例都過不了**，最小反例是 `s1 = "ca"`、`s2 = "bccab"`（正解 `true`）。

### 方法二：維護 `matches` 計數 — O(n1 + n2)／O(1)

方法一每步都重掃 26 格，但**每步其實只有兩個字母的計數會變**。維護 `matches`＝「26 個字母中 `win[c] == need[c]` 的個數」，命中條件變成 `matches == 26`，每步只需修正那兩格的貢獻。

```cpp
// Time: O(n1 + n2)（每步 O(1) 更新 matches）
// Space: O(1)
class Solution {
 public:
  bool checkInclusion(string s1, string s2) {
    int n1 = s1.size(), n2 = s2.size();
    if (n1 > n2) return false;
    array<int, 26> need{}, win{};
    for (int i = 0; i < n1; ++i) {
      ++need[s1[i] - 'a'];
      ++win[s2[i] - 'a'];
    }
    int matches = 0;
    for (int i = 0; i < 26; ++i) matches += (need[i] == win[i]);
    if (matches == 26) return true;

    for (int r = n1; r < n2; ++r) {
      int in = s2[r] - 'a', out = s2[r - n1] - 'a';
      ++win[in];
      if (win[in] == need[in]) ++matches;             // 不等 → 相等
      else if (win[in] == need[in] + 1) --matches;    // 相等 → 不等
      --win[out];
      if (win[out] == need[out]) ++matches;           // 不等 → 相等
      else if (win[out] == need[out] - 1) --matches;  // 相等 → 不等
      if (matches == 26) return true;
    }
    return false;
  }
};
```

> [!important] 為什麼只要檢查兩個邊界值
> `++win[in]` 之後，只有兩種情況會動到 `matches`：正好踩上 `need[in]`（不等變相等），或正好越過成 `need[in] + 1`（代表加之前正好相等，相等變不等）。其餘情況都是「本來就不等、現在還是不等」，貢獻沒變。`--win[out]` 完全對稱。
>
> 進出剛好同一個字母（`in == out`）也不必特判：一加一減會各自觸發一次方向相反的修正，淨值為 0。

> [!note] 該選哪個
> **面試寫方法一。** `n2 ≤ 10⁴` 之下那 26 倍常數完全無感，而且「湊一個定長窗 → 比兩張計數表」是最好口述的說法。
>
> 方法二真正值得記的是 `matches` 這個**增量維護**的手法：本題窗長固定、整表比較還撐得住，但到了窗長會變的 [[0076-Minimum-Window-Substring]]，不維護 `matches`（或 `need` 種類數）就會退化。本題是練這個技巧最乾淨的場地。

兩種寫法都以暴力解（每個起點排序後比對）對拍 84 萬組隨機測資（字母集 1～26、`s1` 長 1～6、`s2` 長 1～12），加上 `n1 > n2`、單字元、`s1` 與 `s2` 等長等邊界案例，全數通過。`n1 = 1000`、`n2 = 10⁵`、字母集 2 的效能：

```txt
  方法一（整表比較）   0.73 ms
  方法二（matches）    0.17 ms
```

## Related Problems

[[0242-Valid-Anagram]] — 本題的靜態版；把「比對兩張 26 格計數表」套進固定長度的滑動窗就是本題
[[0438-Find-All-Anagrams-in-a-String]] — 幾乎同一題，只是把「存在就回傳 true」改成「蒐集所有命中的起點」，同一份 code 改個回傳
[[0424-Longest-Repeating-Character-Replacement]] — 同樣是 26 格計數的滑動窗，但窗長可變、需要 `while` 收縮
[[0076-Minimum-Window-Substring]] — 變長版的計數配對，`matches` 增量維護在那裡從優化變成必需品
[[0003-Longest-Substring-Without-Repeating-Characters]] — 變長窗骨架的起點，對照可看出「定長」省掉了哪些東西
