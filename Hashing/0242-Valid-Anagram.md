---
leetcode-id: 242
difficulty: easy
tags:
  - hash
  - grind-169
  - neetcode-150
solved-1st: 2025-12-09
solved-2nd:
solved-3rd:
memo: 先檢查長度（重要），後扣計數檢查是否中途出現負數
dg-publish: true
---

## Problem Description

Given two strings `s` and `t`, return `true` if `t` is an anagram of `s`, and `false` otherwise.

An anagram is a word or phrase formed by rearranging the letters of a different word or phrase, using all the original letters exactly once.

## Solution

```cpp
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
