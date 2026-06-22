---
leetcode-id: 5
difficulty: medium
tags:
  - dynamic-programming
  - grind-169
  - neetcode-150
memo: Manacher 線性解：插入「#」統一奇偶長度，維護回文邊界 l/r，用對稱位置複用 lps 省去重複擴張，O(n)
dg-publish: true
---

## Problem Description

Given a string `s`, return *the longest* *palindromic* *substring* in `s`.

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
