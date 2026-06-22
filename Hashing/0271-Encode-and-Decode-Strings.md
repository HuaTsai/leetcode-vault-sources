---
leetcode-id: 271
difficulty: medium
tags:
  - hash
  - grind-169
  - neetcode-150
memo: 實作用固定 3 位長度前綴（setw(3) 補零，長度≤999），decode 讀 3 位長度再切；通用法可改「長度#字串」的 length-prefix 避免分隔符誤判
dg-publish: true
---

## Problem Description

Design an algorithm to encode a list of strings to a single string. The encoded string is then decoded back to the original list of strings.

Please implement `encode` and `decode`

## Solution

第一種作法是所有長度先提供，再與字串中間用特殊符號 # 隔開

["abc","de"] -> "3,2#abcde"

第二種作法是每個字串長度固定使用 3 個字元表示（已知字串長度不會超過 999）

["abc","de"] -> "003abc002de"

```cpp
// Time: O(n)
// Space: O(n)
string encode(vector<string> &strs) {
  stringstream ss;
  for (auto s : strs) {
    ss << setw(3) << setfill('0') << to_string(s.size());
    ss << s;
  }
  return ss.str();
}

vector<string> decode(string s) {
  vector<string> ans;
  int i = 0;
  while (i < s.size()) {
    int len = stoi(s.substr(i, 3));
    ans.push_back(s.substr(i + 3, len));
    i += 3 + len;
  }
  return ans;
}
```

第三種作法是通用作法，使用數字搭配符號做區隔

["abc","de"] -> "3#abc2#de"

這種做法的好處是可以處理任意長度的字串，並且有 # 符號做區隔，不會有誤判的問題
