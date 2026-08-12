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

核心觀念：把多個字串接成一條字串，難點在 decode 要能切回來。**不能只靠分隔符**——字串內容本身可能剛好含有該分隔符而造成誤判。正解是 **length-prefix（長度前綴）**：每段前面記下它的長度，decode 時先讀長度、再精確切出那麼多個字元，內容含什麼字元都無所謂。

```txt
["abc", "de"]  →  "3#abc2#de"
                   └┬┘ └┬┘
              讀到 # 前=長度 3，之後取 3 字元 "abc"；再讀 2，取 "de"
```

### 方法一：長度前綴 + 分隔符（通用）— O(n)／O(n)

格式 `長度#內容`。decode 掃到 `#` 前的數字是長度 `len`，`#` 之後 `len` 個字元就是該段內容。`#` 只負責分隔「長度數字」與「內容」，因為切割靠的是長度而非搜尋分隔符，內容裡即使出現 `#` 或數字也不會誤判。任意長度皆可。

```cpp
// Time: O(n)  n 為所有字元總數
// Space: O(n)
string encode(vector<string> &strs) {
  string out;
  for (auto &s : strs) {
    out += to_string(s.size());
    out += '#';
    out += s;
  }
  return out;
}

vector<string> decode(string s) {
  vector<string> res;
  int i = 0;
  while (i < s.size()) {
    int j = i;
    while (s[j] != '#') {
      ++j;  // 找長度數字的結尾
    }
    int len = stoi(s.substr(i, j - i));
    res.push_back(s.substr(j + 1, len));
    i = j + 1 + len;
  }
  return res;
}
```

> [!important]
> 用 length-prefix 而非「純分隔符切割」的理由：字串內容可以是**任意字元序列**（含 `#`、逗號、數字）。只有「先讀長度、再照長度精確切」才能保證無歧義還原。

### 方法二：固定寬度長度前綴 — O(n)／O(n)

每段長度固定用 3 位數表示（`setw(3)` 補零，前提是單段長度 ≤ 999），decode 每次固定讀 3 位。省掉分隔符、解析更簡單，代價是受長度上限限制。

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

> [!warning]
> 固定 3 位版假設**每段長度 ≤ 999**，超過就會截斷長度、解碼錯位；通用版 `長度#內容` 沒有這個上限，較穩健。

## Related Problems

- [[0297-Serialize-and-Deserialize-Binary-Tree]] — 同樣要設計「可逆的自訂編碼」，把結構壓成字串再還原。
- [[0535-Encode-and-Decode-TinyURL]] — 另一種 encode／decode 對稱設計題。
