---
leetcode-id: 125
difficulty: easy
tags:
  - two-pointers
  - grind-169
  - neetcode-150
memo: 左右雙指針相向：用 if-else 鏈先單側跳過非英數字元，再 tolower 比較兩側，不等即 false，O(1) 空間
dg-publish: true
---

## Problem Description

A phrase is a **palindrome** if, after converting all uppercase letters into lowercase letters and removing all non-alphanumeric characters, it reads the same forward and backward. Alphanumeric characters include letters and numbers.

Given a string `s`, return `true` *if it is a **palindrome**, or* `false` *otherwise*.

## Solution

核心觀念：回文即「從兩端往中間，對應字元一路相等」。用相向雙指針 `l`、`r`，遇到非英數字元就單側跳過，兩側都是英數字元時才 `tolower` 後比較；不等即 `false`。不用真的建出「過濾後的新字串」，所以 O(1) 額外空間。

```txt
"A man, a plan, a canal: Panama"
 l→                          ←r
跳過空白/標點，只在兩側都是字母數字時比較：
 a==a  m==m  a==a ...  中間相遇 → true
```

### 方法一：相向雙指針（原地）— O(n)／O(1)

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

> [!warning]
> `isalnum` / `tolower` 的引數若是負值（`char` 在某些平台有號、遇到非 ASCII 位元組會變負）屬未定義行為。嚴謹寫法是轉 `unsigned char`：`isalnum(static_cast<unsigned char>(s[l]))`。本題保證 ASCII 可省，但養成習慣較安全。

> [!tip]
> 用 `if / else if / else` 一次只動一根指針，避免「同時跳兩側」漏掉比較。內層兩側都推進 `++l; --r;` 才是真正比對成功的一步。

### 方法二：過濾後比對反轉 — O(n)／O(n)

先把英數字元濾出、轉小寫成新字串，再判斷它與自己的反轉是否相等。程式碼直觀，代價是 O(n) 額外空間。

```cpp
// Time: O(n)
// Space: O(n)
bool isPalindrome(string s) {
  string f;
  for (char c : s) {
    if (isalnum(static_cast<unsigned char>(c))) {
      f += tolower(static_cast<unsigned char>(c));
    }
  }
  return equal(f.begin(), f.end(), f.rbegin());
}
```

## Related Problems

- [[0167-Two-Sum-II-Input-Array-Is-Sorted]] — 同為相向雙指針的基本款。
- [[0680-Valid-Palindrome-II]] — 進階允許刪一個字元，卡住時分岔嘗試跳左或跳右。
- [[0344-Reverse-String]] — 雙指針原地對調的最簡形。
