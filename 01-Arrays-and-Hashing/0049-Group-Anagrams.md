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

核心觀念：互為 anagram 的字串有一個共通不變量——**排序後相同**，或**每個字母的出現次數相同**。只要為每個字串算出一個「同組必相同」的 canonical key，用 `unordered_map<key, 該組字串>` 一趟就能分好組，把「兩兩比對是否 anagram」的 O(n²) 降成 O(n)。

設 `n` = 字串數，`k` = 字串最大長度。

### 方法一：排序字串當 key — O(n·k·log k)／O(n·k)

最直觀：把每個字串排序後的結果當 key，anagram 排序後必然相同，自然落到同一桶。

```cpp
// Time: O(n * k log k)
// Space: O(n * k)
vector<vector<string>> groupAnagrams(vector<string> &strs) {
  unordered_map<string, vector<string>> mp;
  for (auto s : strs) {
    auto s2 = s;
    ranges::sort(s2);
    mp[s2].push_back(s);  // 排序後字串當 key
  }
  vector<vector<string>> ans;
  for (auto [k, v] : mp) {
    ans.push_back(v);
  }
  return ans;
}
```

### 方法二：字母計數當 key（避開排序）— O(n·k)／O(n·k)

用長度 26 的計數陣列組出 key，省掉每個字串的排序 log 因子，是漸進最優解。

```cpp
// Time: O(n * k)
// Space: O(n * k)
vector<vector<string>> groupAnagrams(vector<string> &strs) {
  unordered_map<string, vector<string>> mp;
  for (auto s : strs) {
    vector<int> cnt(26);
    for (auto c : s) {
      ++cnt[c - 'a'];
    }
    string key = "|";
    for (auto n : cnt) {
      key += to_string(n) + "|";  // 分隔符不可省
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

> [!warning]
> 計數 key 一定要加分隔符。若直接把數字接起來，`a×1,b×11` 會得到 `"1","11"` 拼成 `111`，而 `a×11,b×1` 也是 `111` → 兩個不同組被誤判為同組。加 `|`（或 `#`）分隔即可消除歧義。

### 方法三：線性掃描比對（自己的初版，較慢）— O(n²·k)

不用 hash map，維護一個「各組 canonical 字串」清單，每個新字串排序後線性掃描找相符的組。邏輯正確但每個字串都要掃過所有已知組，量級大時容易 TLE。

```cpp
// Time: O(n^2 * k)
// Space: O(n * k)
vector<vector<string>> groupAnagrams(vector<string> &strs) {
  vector<vector<string>> ans;
  vector<string> keys;  // keys[i] = ans[i] 這組的排序後字串
  for (auto str : strs) {
    auto sorted = str;
    ranges::sort(sorted);
    bool found = false;
    for (int i = 0; i < ans.size(); ++i) {
      if (sorted == keys[i]) {
        ans[i].push_back(str);
        found = true;
        break;
      }
    }
    if (!found) {
      ans.push_back({str});
      keys.push_back(sorted);
    }
  }
  return ans;
}
```

> [!tip]
> 三者的差別只在「怎麼找到同組」：方法三線性掃 O(n) 找組、方法一/二用 hash O(1) 找組。key 一旦設計成「同組必相同」，換成 hash 就是自然的優化。

## Related Problems

- [[0242-Valid-Anagram]] — 判斷單一一對是否 anagram，本題的基礎；同樣的 canonical key 思想。
- [[0249-Group-Shifted-Strings]] — 換一種分組規則（位移序列），練習設計 canonical key。
- [[0438-Find-All-Anagrams-in-a-String]] — anagram + 滑動視窗，找子字串。
