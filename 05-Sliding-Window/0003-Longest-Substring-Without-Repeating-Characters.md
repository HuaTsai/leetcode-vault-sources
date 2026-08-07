---
leetcode-id: 3
difficulty: medium
tags:
  - hash
  - sliding-window
  - string
  - grind-169
  - neetcode-150
memo: 維持「窗口內無重複」的不變量，右界每前進一格就從左邊縮到重複消失，每個字元進出各一次故均攤 O(n)；易錯在拿字元直接當索引，char 是有號的、非 ASCII 位元組會變負索引而越界
dg-publish: true
---

## Problem Description

Given a string `s`, find the length of the longest substring without duplicate characters.

A substring is a contiguous non-empty sequence of characters within a string.

Constraints:

- `s` consists of English letters, digits, symbols and spaces.

## Solution

核心觀念：維持一個**內部無重複**的窗口 `s[l..r]`。右界每前進一格，如果新字元已經在窗口裡，就從左邊逐格移除、直到那個重複消失。整個過程中 `l` 只會前進不會後退，所以每個字元最多進出窗口各一次——**均攤 O(n)**，雖然裡面有個 `while`。

```txt
s = a b c a b c b b
    0 1 2 3 4 5 6 7

 r  s[r]  動作                             窗口        長度
 0   a    直接加入                         [0,0] a       1
 1   b    直接加入                         [0,1] ab      2
 2   c    直接加入                         [0,2] abc     3  ← 答案
 3   a    重複，移除 s[0]=a，l=1           [1,3] bca     3
 4   b    重複，移除 s[1]=b，l=2           [2,4] cab     3
 5   c    重複，移除 s[2]=c，l=3           [3,5] abc     3
 6   b    重複，移除 s[3]=a、s[4]=b，l=5   [5,6] cb      2
 7   b    重複，移除 s[5]=c、s[6]=b，l=7   [7,7] b       1
```

> [!important] 不變量：迴圈每輪結束時 `s[l..r]` 內部無重複
> 所有滑動窗口題都是同一個骨架，**差別只在收縮條件**：
>
> ```txt
> for (r = 0; r < n; ++r) {
>   加入 s[r]
>   while (窗口不合法) { 移除 s[l]; ++l; }
>   更新答案
> }
> ```
>
> 本題的「不合法」是「有重複」；[[0424-Longest-Repeating-Character-Replacement]] 是「窗口長度 − 出現最多的字元數 > k」；[[0076-Minimum-Window-Substring]] 是「已覆蓋目標」（但那題求最小，所以是**合法時才收縮**）。**認出骨架，換題只要換那一行判斷。**

### 方法一：逐格收縮 — O(n)／O(1)（推薦）

```cpp
// Time: O(n) 均攤（每個字元進出窗口各一次）
// Space: O(1)（固定 256 格，與輸入長度無關）
class Solution {
 public:
  int lengthOfLongestSubstring(string s) {
    array<bool, 256> used{};
    int ans = 0, n = s.size();
    for (int l = 0, r = 0; r < n; ++r) {
      while (used[(unsigned char)s[r]]) {      // 窗口內已有 s[r]，從左邊縮到它消失
        used[(unsigned char)s[l]] = false;
        ++l;
      }
      used[(unsigned char)s[r]] = true;
      ans = max(ans, r - l + 1);
    }
    return ans;
  }
};
```

把 `l`、`r` 都宣告在 `for` 的初始化區，兩個指標的生命週期就跟窗口一致，不會外洩到迴圈外。

> [!warning] `s[r]` 直接當索引會越界
> `std::string::operator[]` 回傳 **`char`**，而 x86-64 Linux 上 `char` 是**有號**的。位元組值 `≥ 128` 會變成負數：
>
> ```txt
> s = "a" + UTF-8 的「中」(E4 B8 AD)
>   s[1] 當 char = -28    當 unsigned char = 228
>   s[2] 當 char = -72    當 unsigned char = 184
> ```
>
> `used[-28]` 就是負索引存取。實測用 AddressSanitizer 跑：
>
> ```txt
> ERROR: AddressSanitizer: heap-buffer-overflow
> SUMMARY: ... in std::_Bit_reference::operator bool() const
> ```
>
> 本題 constraints 寫明只有 English letters、digits、symbols、spaces（全 ASCII），所以不轉型也能 AC。但轉型是免費的，**任何拿字元當陣列索引的地方都該寫 `(unsigned char)`**。

> [!tip] 別用 `vector<bool>` 當查表
> `vector<bool>` 是**位元打包的特化**，`operator[]` 回傳 proxy 而非 `bool&`，每次存取都要位移加遮罩。300 萬字元的字串實測：
>
> ```txt
> 字元集大小 26：
>   vector<bool>  15.5 ms
>   array<bool>   12.4 ms      快 20%
> ```
>
> 而且 `vector<bool> used(256)` 每次呼叫都會做一次堆積配置，`array<bool, 256> used{}` 在堆疊上零初始化。詳見 [[STL-Pitfalls]]。

### 方法二：記住上次出現位置，左界直接跳 — O(n)／O(1)

不逐格退，改記錄每個字元**上次出現的下標**，一步跳到位：

```cpp
// Time: O(n)（每步 worst-case O(1)，不需要均攤論證）
// Space: O(1)
class Solution {
 public:
  int lengthOfLongestSubstring(string s) {
    array<int, 256> last;
    last.fill(-1);
    int ans = 0, n = s.size();
    for (int l = 0, r = 0; r < n; ++r) {
      unsigned char c = s[r];
      if (last[c] >= l) l = last[c] + 1;       // 只有落在窗口內才跳
      last[c] = r;
      ans = max(ans, r - l + 1);
    }
    return ans;
  }
};
```

兩者都是 O(n)，但常數差很多。300 萬字元實測：

```txt
                        字元集 4    字元集 26   字元集 128
  方法一（逐格縮）        20.0 ms     15.5 ms      10.8 ms
  方法二（直接跳）         1.8 ms      1.8 ms       1.8 ms
```

**快 6～11 倍，而且耗時與字元集大小無關。**方法一的字元集越小、重複越密集、收縮就越頻繁；方法二每輪固定一次比較加一次寫入，沒有內層迴圈。

> [!warning] `if (last[c] >= l)` 這個守衛不能省
> `last[c]` 記的是**全域**上次出現位置，可能落在窗口**左邊**——那個字元早就離開窗口了，不該讓它把 `l` 往回拉。無條件寫 `l = last[c] + 1` 的話：
>
> ```txt
> "abba" → 回傳 3（正解 2）
> 20 萬組隨機測資錯 102462 組
> ```
>
> 寫成 `l = max(l, last[c] + 1)` 也正確，兩種守衛等價。

> [!note] 該選哪個
> **方法一是首選**，儘管慢。理由是它的形狀可以直接搬到其他滑動窗口題（只換收縮條件），而且 `l` 結構上只會前進，**不存在方法二那個守衛陷阱**。方法二快，但它把「窗口左界」和「字元歷史位置」兩個概念耦合在一起，換題就得重想。
>
> LeetCode 的資料量下兩者都是瞬間完成——**選好維護、可遷移的那個**。

兩種寫法實測（30 萬組隨機測資 + `""`、`"abba"`、`"dvdf"`、`"pwwkew"` 等邊界，並在 ASan／UBSan 下以非 ASCII 位元組驗證）全數通過。

## Related Problems

[[0424-Longest-Repeating-Character-Replacement]] — 同一個骨架，收縮條件換成「窗口長度 − 最多字元出現次數 > k」
[[0076-Minimum-Window-Substring]] — 同骨架但求**最小**，所以是窗口合法時才收縮，收縮時機剛好相反
[[0567-Permutation-In-String]] — 窗口長度固定，退化成不需要 `while` 的定長滑動
[[0121-Best-Time-to-Buy-and-Sell-Stock]] — 同資料夾的對照組，那題的左界是「直接跳」而非逐格縮
[[STL-Pitfalls]] — 本題踩到的 `vector<bool>` 位元打包與 `char` 有號性都收在該篇第五類，另附 sanitizer 旗標組合
