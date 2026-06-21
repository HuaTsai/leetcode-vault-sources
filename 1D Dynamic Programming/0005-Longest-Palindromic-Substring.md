---
leetcode-id: 5
difficulty: medium
tags:
  - dynamic-programming
  - grind-169
  - neetcode-150
solved-1st: 2025-12-12
solved-2nd:
solved-3rd:
memo: 中間字元左右擴張、預處理技巧、Manacher 演算法
dg-publish: true
---

## Problem Description

Given a string `s`, return _the longest_ _palindromic_ _substring_ in `s`.

## Solution

```cpp
string longestPalindrome(string s0) {
  std::string s = "#";
  for (char c : s0) {
    s += c;
    s += "#";
  }
  int l = 0;
  int r = 0;
  int n = s.size();
  int max_pos = 0;
  vector<int> lps(n);
  for (int i = 0; i < n; ++i) {
    if (i <= r) {
      lps[i] = min(lps[l + (r - i)], r - i);
    }

    while (i - lps[i] - 1 >= 0 && i + lps[i] + 1 < n &&
           s[i - lps[i] - 1] == s[i + lps[i] + 1]) {
      ++lps[i];
      l = i - lps[i];
      r = i + lps[i];
    }

    if (lps[i] > lps[max_pos]) {
      max_pos = i;
    }
  }
  return s0.substr((max_pos - lps[max_pos]) / 2, lps[max_pos]);
}
```

## Related Problems

[[1111-Longest-Palindrome]]
