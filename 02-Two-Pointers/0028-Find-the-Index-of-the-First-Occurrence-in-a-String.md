---
leetcode-id: 28
difficulty: easy
tags:
  - two-pointers
  - string
  - string-matching
  - knuth-morris-pratt
  - z-algorithm
  - boyer-moore
memo: KMP 失配時 j 退到 lps 值續比而 i 永不回退；建表是 needle 錯位自比，與主匹配共用同一套退位骨架；建表的退位 while 結束後要補一次相等判斷（三種情況裡只有兩種要加一），省掉就把「退到底才接上」誤判成 0
dg-publish: true
---

## Problem Description

Given two strings `needle` and `haystack`, return the index of the first occurrence of `needle` in `haystack`, or `-1` if `needle` is not part of `haystack`.

## Solution

核心觀念：暴力法失配後把起點右移一格、從頭重比，等於把「剛才已經比對過的那段」的資訊全部丟掉。KMP 把這份資訊預先存進 **LPS 表**（Longest proper Prefix which is also Suffix）：`lps[i]` 是 `needle[0..i]` 的最長「真前綴＝真後綴」長度。匹配到 `j` 失配時，已匹配的前 `j` 個字元中，長 `lps[j-1]` 的前綴同時也是後綴——把 `j` 退到 `lps[j-1]` 就等於讓 needle 一次滑到下一個可能對齊的位置，而文本指標 `i` **永不回退**。`j` 每輪最多加一、退位總量不會超過前進總量，所以整體是攤銷 O(n+m)。

```txt
needle:   a a b a a a c
index:    0 1 2 3 4 5 6
lps:      0 1 0 1 2 2 0    lps[i] 是 needle[0..i] 的最長「真前綴＝真後綴」長度
                ─ ─        例：lps[5]=2，因為 "aabaaa" 的頭尾都是 "aa"

haystack:  a a b a a a b a a a c
           │ │ │ │ │ │ ✗            i=6, j=6 失配（'b' ≠ 'c'）
needle:    a a b a a a c            已匹配 "aabaaa"，最長真前綴＝真後綴是 "aa"
                   ─ ─
           j 退到 lps[5]=2、i 不動 ⇒ 等價於 needle 一次右移 4 格
needle:            a a b a a a c
                   ─ ─ ↑
           已匹配的 "aa" 免重比，從 j=2 接著比 haystack[6]，最後在 4 找到答案
```

> [!important] 建表退位迴圈結束後，必須補一次相等判斷
> `while (len > 0 && str[i] != str[len])` 結束時有三種情況：
>
> 1. `len == 0` 且 `str[i] != str[0]` — 完全接不上，`lps[i] = 0`
> 2. `len == 0` 且 `str[i] == str[0]` — 退到底才接上，`lps[i] = 1`
> 3. `len > 0` 且 `str[i] == str[len]` — 中途接上，`lps[i] = len + 1`
>
> 只有 2、3 要 `len++`，所以 while 後那句 `if (str[i] == str[len]) len++;` 不能省——省掉就把情況 2 誤判成 0。

### 方法一：KMP — O(n+m)／O(m)

```cpp
// Time: O(n+m)  建表 O(m)；主迴圈 i 永不回退，j 退位總量 ≤ 前進總量，攤銷 O(n)
// Space: O(m)   LPS 表
class Solution {
 public:
  vector<int> BuildLPS(const string &str) {
    int n = str.size();
    vector<int> lps(n);
    for (int i = 1; i < n; ++i) {
      int len = lps[i - 1];
      while (len > 0 && str[i] != str[len]) {
        len = lps[len - 1];
      }
      if (str[i] == str[len]) {
        len++;  // 三種情況裡只有「接得上」的兩種要加一
      }
      lps[i] = len;
    }
    return lps;
  }

  int strStr(string haystack, string needle) {
    int n = haystack.size();
    int m = needle.size();
    if (m == 0) return 0;  // 慣例：空 pattern 匹配於 0（本題保證 m ≥ 1）
    auto lps = BuildLPS(needle);
    for (int i = 0, j = 0; i < n; ++i) {
      while (j > 0 && haystack[i] != needle[j]) {
        j = lps[j - 1];
      }
      if (haystack[i] == needle[j]) {
        ++j;
      }
      if (j == m) {
        return i - m + 1;
      }
    }
    return -1;
  }
};
```

> [!tip] 建表與匹配是同一套骨架
> `BuildLPS` 是「needle 錯位跟自己比」（`str[i]` 對 `str[len]`），`strStr` 是「haystack 跟 needle 比」（`haystack[i]` 對 `needle[j]`）：同樣是失配就退到 `lps[·-1]`、相等就前進、計數到 m 就收割。把 `len` 改名成 `j`，兩段程式幾乎逐行同構——記熟一份等於記熟兩份。

### 方法二：KMP（while 三分支寫法）— O(n+m)／O(m)

同一演算法換一種控制流：三個分支互斥——相等就雙雙前進、退得動就退位（文本指標不動）、退無可退才只推進文本指標。每一步只做一件事，建表與匹配的對稱性更一目了然；終止性靠的是退位分支讓 `len`（或 `j`）嚴格遞減，總會落入另外兩個分支之一。

```cpp
// Time: O(n+m)  同方法一，僅控制流不同
// Space: O(m)
class Solution {
 public:
  vector<int> BuildLPS(const string &str) {
    int n = str.size();
    vector<int> lps(n);
    int i = 1, len = 0;
    while (i < n) {
      if (str[i] == str[len]) {
        lps[i++] = ++len;  // 接得上：本格確定，雙雙前進
      } else if (len > 0) {
        len = lps[len - 1];  // 退位再試，i 不動
      } else {
        lps[i++] = 0;  // 退無可退：本格為 0
      }
    }
    return lps;
  }

  int strStr(string haystack, string needle) {
    int n = haystack.size();
    int m = needle.size();
    if (m == 0) return 0;
    auto lps = BuildLPS(needle);
    int i = 0, j = 0;
    while (i < n) {
      if (haystack[i] == needle[j]) {
        ++i, ++j;  // 匹配：雙雙前進
        if (j == m) return i - m;
      } else if (j > 0) {
        j = lps[j - 1];  // 退位再試，i 不動
      } else {
        ++i;  // j 已在 0，只能推進文本
      }
    }
    return -1;
  }
};
```

> [!warning] 兩種寫法的收割點差一
> for 版在 `++j` 之後、`++i` 之前檢查 `j == m`，回傳 `i - m + 1`；while 版在 `++i, ++j` **都做完**之後才檢查，回傳 `i - m`。混抄兩版最容易錯在這一格。

### 方法三：KMP 串接技巧 — O(n+m)／O(n+m)

LPS 的定義本身就是一種「匹配」：對 `s = needle + '#' + haystack` 建表，位置 `k` 的 lps 值等於 `m`，代表以 `k` 結尾的 `m` 個字元恰好是整條 needle。於是完全不用寫匹配迴圈，一個 `BuildLPS` 打天下。索引換算：`k` 對應 haystack 的結尾索引 `k - (m+1)`，起點再往前 `m - 1` 格，即 `k - 2m`。

```cpp
// Time: O(n+m)   對長度 n+m+1 的串接字串建一次表
// Space: O(n+m)  串接字串與其 LPS 表
// BuildLPS 同方法一
class Solution {
 public:
  int strStr(string haystack, string needle) {
    int m = needle.size();
    if (m == 0) return 0;
    string s = needle + '#' + haystack;
    auto lps = BuildLPS(s);
    for (int k = m + 1; k < (int)s.size(); ++k) {
      if (lps[k] == m) {
        return k - 2 * m;  // 結尾在 haystack 的 k-(m+1)，起點再前移 m-1
      }
    }
    return -1;
  }
};
```

> [!warning] 分隔字元必須不出現在兩條字串的字元集裡
> 少了 `'#'`（或選到可能出現的字元），lps／z 值就可能跨過邊界長過 `m`，把「needle 前綴＋分隔＋haystack 開頭」誤算成一段匹配。有了域外分隔字元，任何跨界匹配都得吃下那個字元而必然失配，`lps[k] ≤ m` 才有保證。本題僅小寫字母，`'#'` 安全；方法四同理。

### 方法四：Z-Algorithm — O(n+m)／O(n+m)

`z[i]` 是 `s` 從 `i` 開始的後綴與 `s` 整體前綴的最長公共前綴長度。演算法維護延伸最右的 **Z-box** `[l, r)`：新位置若落在 box 內，先抄對稱位置的答案 `min(r - i, z[i - l])`，出了 box 才逐字硬比；`r` 只增不減，所以攤銷線性。同樣串接 `needle + '#' + haystack`，`z[k] == m` 即匹配。與方法三互為對偶——`lps[k]` 問「**以 k 結尾**的那段是不是前綴」，`z[k]` 問「**從 k 開始**的那段是不是前綴」，所以 Z 版的起點就是 `k` 對應的位置 `k - (m+1)`，不用再減。

```cpp
// Time: O(n+m)   r 只增不減，內層 while 每走一步 r 就加一，攤銷線性
// Space: O(n+m)  串接字串與 z 表
class Solution {
 public:
  vector<int> ZFunction(const string &s) {
    int n = s.size();
    vector<int> z(n);
    for (int i = 1, l = 0, r = 0; i < n; ++i) {
      if (i < r) {
        z[i] = min(r - i, z[i - l]);  // 在 Z-box 內：先抄前面算過的答案
      }
      while (i + z[i] < n && s[z[i]] == s[i + z[i]]) {
        ++z[i];  // 出了 box 才逐字硬比
      }
      if (i + z[i] > r) {
        l = i, r = i + z[i];  // 延伸出更右的 box 就更新
      }
    }
    return z;
  }

  int strStr(string haystack, string needle) {
    int m = needle.size();
    if (m == 0) return 0;
    string s = needle + '#' + haystack;
    auto z = ZFunction(s);
    for (int k = m + 1; k < (int)s.size(); ++k) {
      if (z[k] == m) {
        return k - m - 1;  // z 量測「從 k 開始」，起點就是 k 對應的 haystack 位置
      }
    }
    return -1;
  }
};
```

### 方法五：暴力法 — O(n·m)／O(1)

n、m ≤ 10⁴，最壞約 10⁸ 次字元比較，本題其實過得了。價值在對照：暴力法失配後起點右移一格、`j` 歸零重比，KMP 省掉的正是這個回退——已匹配段落攜帶的資訊被 LPS 表留了下來。

```cpp
// Time: O(n·m)  每個起點最壞比 m 個字元
// Space: O(1)
class Solution {
 public:
  int strStr(string haystack, string needle) {
    int n = haystack.size();
    int m = needle.size();
    for (int i = 0; i + m <= n; ++i) {
      if (haystack.compare(i, m, needle) == 0) {
        return i;
      }
    }
    return -1;
  }
};
```

> [!note] 空 needle 與 Boyer-Moore
>
> - 本題保證 `needle.length ≥ 1`，但慣例上（C 的 `strstr`、Java 的 `indexOf`）空 pattern 應回傳 0。KMP 版不加 guard 時 m = 0 會回傳 1——`j == m` 的檢查落在 i = 0 那輪的**尾端**，回傳 `i - m + 1 = 1`——所以各版本開頭補了 `if (m == 0) return 0;`（方法五的 `i + m <= n` 天生正確）。
> - **Boyer-Moore** 是實務文本搜尋（grep 類工具）的主流：從 pattern **尾端**往前比，靠壞字元／好後綴表一次跳過整段文本，平均次線性；但表格出名地難寫對、純壞字元版最壞仍是 O(n·m)，刷題掌握 KMP／Z 就夠。C++17 起可直接用 `std::search(h.begin(), h.end(), boyer_moore_searcher(nd.begin(), nd.end()))`。

## Related Problems

- [[1392-Longest-Happy-Prefix]] — 答案就是整條字串 LPS 表的最後一格
- [[0459-Repeated-Substring-Pattern]] — 用 LPS 判斷字串是否由重複子串構成（n − lps[n−1] 是最小週期）
- [[0214-Shortest-Palindrome]] — 串接技巧的變形：s + '#' + reverse(s) 建表找最長回文前綴
- [[0686-Repeated-String-Match]] — strStr 的直接應用，重點在估出重複次數的上界
