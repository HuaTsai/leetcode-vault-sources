---
leetcode-id: 49
difficulty: medium
tags:
  - hash
  - grind-169
  - neetcode-150
memo: 1. 排序的字串當作鍵值 2. 可設計逗號區隔的特殊鍵值
dg-publish: true
---

## Problem Description

Given an array of strings strs, group the anagrams together. You can return the answer in any order.

- 1 <= strs.length <= 10^4
- 0 <= strs[i].length <= 100

## Solution

即使量級不大，仍然可能造成 TLE

速度偏慢：

```cpp
vector<vector<string>> groupAnagrams(vector<string> &strs) {
  if (strs.empty()) {
    return {{""}};
  }
  vector<vector<string>> ans;
  vector<string> sorted;
  for (auto str : strs) {
    auto str2 = str;
    ranges::sort(str2);
    bool found = false;
    for (int i = 0; i < ans.size(); ++i) {
      if (str2 == sorted[i]) {
        ans[i].push_back(str);
        found = true;
        break;
      }
    }
    if (!found) {
      ans.push_back({str});
      ranges::sort(str);
      sorted.push_back(str);
    }
  }
  return ans;
}
```

較優雅的解法：可以把排序後的字串當 key，使用 hash map 來存放結果

```cpp
vector<vector<string>> groupAnagrams(vector<string> &strs) {
  unordered_map<string, vector<string>> mp;
  for (auto s : strs) {
    auto s2 = s;
    ranges::sort(s2);
    mp[s2].push_back(s);
  }
  vector<vector<string>> ans;
  for (auto [k, v] : mp) {
    ans.push_back(v);
  }
  return ans;
}
```

設計特殊鍵值的解法（避免排序）：

```cpp
vector<vector<string>> groupAnagrams(vector<string> &strs) {
  unordered_map<string, vector<string>> mp;
  for (auto s : strs) {
    vector<int> cnt(26);
    for (auto c : s) {
      ++cnt[c - 'a'];
    }
    std::string key = "|";
    for (auto n : cnt) {
      key += to_string(n) + "|";
    }
    mp[key].push_back(s);
  }
  vector<vector<string>> ans;
  for (auto [k, v] : mp) {
    ans.push_back(v);
  }
  return ans;
}
```

## Related Problems

[[0242-Valid-Anagram]]
