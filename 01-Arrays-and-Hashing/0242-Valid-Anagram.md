---
leetcode-id: 242
difficulty: easy
tags:
  - hash
  - grind-169
  - neetcode-150
memo: 先檢查長度（重要），後扣計數檢查是否中途出現負數
dg-publish: true
---

## Problem Description

Given two strings `s` and `t`, return `true` if `t` is an anagram of `s`, and `false` otherwise.

An anagram is a word or phrase formed by rearranging the letters of a different word or phrase, using all the original letters exactly once.

## Solution

核心觀念：anagram 的定義就是「兩字串每個字母出現次數完全相同」。長度不同必不成立，先擋掉；接著用長度 26 的計數陣列，掃 `s` 全部 `+1`、掃 `t` 全部 `-1`，過程中一旦某字母被扣成負數就代表 `t` 比 `s` 多用了它，立刻 `false`。全程走完沒出事即為 anagram。

### 方法一：計數陣列一加一減 — O(n)／O(1)

```cpp
// Time: O(n)
// Space: O(1)  （固定 26 格）
bool isAnagram(string s, string t) {
  if (s.size() != t.size()) {
    return false;
  }
  vector<int> letters(26);
  for (char c : s) {
    ++letters[c - 'a'];
  }
  for (char c : t) {
    if (--letters[c - 'a'] < 0) {
      return false;
    }
  }
  return true;
}
```

> [!important]
> **先比長度**再進迴圈：長度不同時直接 `false`，也讓「加一減一後陣列必歸零」的推論成立——長度相等且無負數，就保證每格恰好抵消，不必再掃一次確認全為 0。

> [!note]
> 只用一個計數陣列（加減同一份）就夠；若題目擴展到 Unicode，把 `vector<int>(26)` 換成 `unordered_map<char,int>` 即可，邏輯不變。

### 方法二：排序後比較 — O(n log n)／O(1)

把兩字串各自排序，若相等即為 anagram。寫法最短，但多了排序的 log 因子。

```cpp
// Time: O(n log n)
// Space: O(1)  （in-place 排序，不計輸出）
bool isAnagram(string s, string t) {
  if (s.size() != t.size()) {
    return false;
  }
  ranges::sort(s);
  ranges::sort(t);
  return s == t;
}
```

## Related Problems

- [[0049-Group-Anagrams]] — 把「是否 anagram」推廣成「把所有 anagram 分組」，用計數字串當 key。
- [[0217-Contains-Duplicate]] — 同屬用 hash／計數判斷組成的入門題。
- [[0383-Ransom-Note]] — 幾乎相同的計數扣減，只是改判「t 是否夠湊出 s」。
