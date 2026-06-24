---
leetcode-id: 20
difficulty: easy
tags:
  - stack
  - string
  - grind-169
  - neetcode-150
memo: 括號匹配本質是 LIFO 用 stack；訣竅是遇左括號就壓入「對應的右括號」，遇右括號直接比對棧頂，最後務必檢查 stack 為空，否則 ((( 這類未閉合會被漏判
dg-publish: true
---

## Problem Description

Given a string s containing just the characters `(`, `)`, `{`, `}`, `[` and `]`, determine if the input string is valid.

An input string is valid if:

1. Open brackets must be closed by the same type of brackets.
2. Open brackets must be closed in the correct order.
3. Every close bracket has a corresponding open bracket of the same type.

核心觀念：括號匹配本質是 **LIFO**——「最近打開的括號，必須最先被關閉」，這正是 stack 的語義。從左到右掃描，左括號代表「開了一個待關閉的承諾」就壓進去；右括號代表「要兌現承諾」，必須對應到**棧頂**那個最近的左括號。任何時候對不上、或掃完還有沒兌現的承諾，字串就不合法。

字串不合法只會從三個地方冒出來，對應三個 `false`：

```txt
1. 右括號但 stack 是空的     ")"     沒有對應的左括號
2. 右括號但棧頂類型不符      "([)]"  最近打開的是 [，卻想用 ) 關
3. 掃完字串 stack 還非空     "((("   有左括號從沒被關閉
```

### 方法一：壓入「對應的右括號」 — O(n)／O(n)（推薦）

遇到左括號時，不存它本身，而是直接把**期待之後出現的右括號**壓進去。如此一來，遇到任何右括號只要跟棧頂做一次相等比較即可，省掉三組「哪個配哪個」的分支判斷，邏輯最精簡。

```cpp
// Time: O(n)
// Space: O(n)  最壞全是左括號，stack 存滿
class Solution {
 public:
  bool isValid(string s) {
    stack<char> stk;
    for (char c : s) {
      if (c == '(') stk.push(')');
      else if (c == '[') stk.push(']');
      else if (c == '{') stk.push('}');
      else if (stk.empty() || stk.top() != c) return false;  // 空或對不上
      else stk.pop();                                        // 對上了，兌現
    }
    return stk.empty();  // 必須檢查：還有沒兌現的左括號就不合法
  }
};
```

> [!tip] 把「配對表」搬進 stack
> 一般直覺是壓左括號、彈出時再判斷 `top` 該配哪個右括號。反過來壓「期待的右括號」後，比對退化成單純的 `stk.top() != c`，不需要任何映射或多重 `||`。這是括號類題目很常用的小技巧。

### 方法二：壓左括號，彈出時比對類型 — O(n)／O(n)

你原本的作法：照字面壓入左括號，遇右括號時先確認非空，再彈出棧頂、逐一比對三種配對。語義最直白，正確性一目了然。

```cpp
// Time: O(n)
// Space: O(n)
class Solution {
 public:
  bool isValid(string s) {
    stack<char> stk;
    for (auto c : s) {
      if (c == '(' || c == '[' || c == '{') {
        stk.push(c);
        continue;
      }
      if (stk.empty()) {
        return false;
      }
      auto top = stk.top();
      stk.pop();
      if ((c == ')' && top != '(') ||
          (c == ']' && top != '[') ||
          (c == '}' && top != '{')) {
        return false;
      }
    }
    return stk.empty();
  }
};
```

> [!tip] 三組 `||` 可收成查表
> 那串 `(c == ')' && top != '(') || ...` 可用一張 `unordered_map<char, char>` 把右括號映射到對應左括號，比對收成一行：
>
> ```cpp
> unordered_map<char, char> match = {{')', '('}, {']', '['}, {'}', '{'}};
> for (char c : s) {
>   if (!match.contains(c)) stk.push(c);              // 左括號
>   else if (stk.empty() || stk.top() != match[c]) return false;
>   else stk.pop();
> }
> ```

> [!warning] 別漏掉最後的 `stk.empty()`
> 迴圈跑完只代表「沒有任何右括號對不上」，不代表合法——`"((("` 全程不會觸發任何 `false`，但結束時 stack 還有三個左括號。少了結尾的 `return stk.empty()`、直接 `return true`，這類「只開不關」的字串就會被誤判成合法。另外彈出前一定要先確認 `!stk.empty()`，對空 stack 呼叫 `top()` / `pop()` 是未定義行為。

## Related Problems

[[0022-Generate-Parentheses]] — 反向題：用 backtracking 生成所有合法括號組合，底層的合法性規則與本題同源
[[0032-Longest-Valid-Parentheses]] — 進階：stack 改存 index，求最長合法括號子字串
[[0150-Evaluate-Reverse-Polish-Notation]] — 同樣用 stack 解析表達式，遇運算子就彈出運算元
[[0155-Min-Stack]] — stack 基礎操作題，本題的前置暖身
[[0735-Asteroid-Collision]] — stack 處理「後到元素與棧頂互相抵銷」，與括號成對抵銷同構
