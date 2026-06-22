---
leetcode-id: 125
difficulty: easy
tags:
  - two-pointers
  - grind-169
  - neetcode-150
memo: 雙指針基礎應用，可使用 while + if-else 解
dg-publish: true
---

## Problem Description

A phrase is a **palindrome** if, after converting all uppercase letters into lowercase letters and removing all non-alphanumeric characters, it reads the same forward and backward. Alphanumeric characters include letters and numbers.

Given a string `s`, return `true` _if it is a **palindrome**, or_ `false` _otherwise_.

## Solution

```cpp
// Time: O(n)
// Space: O(1)
bool isPalindrome(string s) {
  int l = 0;
  int r = s.size() - 1;
  while (l < r) {
    if (!isalnum(s[l])) {
      ++l;
    } else if (!isalnum(s[r])) {
      --r;
    } else {
      if (tolower(s[l]) != tolower(s[r])) {
        return false;
      }
      ++l;
      --r;
    }
  }
  return true;
}
```
