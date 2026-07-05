---
leetcode-id: 150
difficulty: medium
tags:
  - stack
  - array
  - math
  - grind-169
  - neetcode-150
memo: 遇數字壓棧、遇運算子彈兩個算完壓回；先彈出的是右運算元，減除順序寫反就錯，且 C++ 整數除法天然朝零截斷
dg-publish: true
---

## Problem Description

You are given an array of strings tokens that represents an arithmetic expression in a Reverse Polish Notation.

**_Reverse Polish Notation_** is a mathematical notation in which every operator follows all of its operands. For example, to add three and four, one would write "3 4 +" rather than "3 + 4". If there are multiple operations, the operator is given immediately after its second operand; so the expression written "3 − 4 + 5" would be written "3 4 − 5 +" first subtract 4 from 3, then add 5 to that.

Evaluate the expression. Return an integer that represents the value of the expression.

Note that:

- The valid operators are '+', '-', '\*', and '/'.
- Each operand may be an integer or another expression.
- The division between two integers always truncates toward zero.
- There will not be any division by zero.
- The input represents a valid arithmetic expression in a reverse polish notation.
- The answer and all the intermediate calculations can be represented in a 32-bit integer.

## Solution

核心觀念：RPN 的設計讓運算子**永遠出現在它的兩個運算元之後**，所以不需要括號、也不需要優先級——從左到右掃一遍，遇到數字就壓進 stack「先存著」，遇到運算子就代表「它的兩個運算元已經備妥在棧頂」，彈出兩個、算完把結果壓回去，這個結果又成為外層運算的運算元。掃完後棧裡剩下的唯一元素就是答案。

```txt
tokens:  2      1      +       3      *
stack:  [2]   [2,1]   [3]    [3,3]   [9]
                       ↑ 彈出 1、2 → 2+1=3 壓回
```

> [!important] RPN 就是表達式樹的後序走訪
> 中序 `(2+1)*3` 需要括號與優先級才能還原結構；後序 `2 1 + 3 *` 把結構攤平在順序裡。stack 求值等價於把表達式樹**自底向上收合**：每遇到一個運算子節點，它的兩個子樹恰好是棧頂那兩個已算好的值。

### 方法一：遇運算子彈兩個、其餘壓棧 — O(n)／O(n)（推薦）

最正規的寫法：整字串比對判斷是否為運算子，四個分支直接運算。沒有任何抽象層，面試時最快寫對、也最快跑完。

```cpp
// Time: O(n)  每個 token 恰好處理一次
// Space: O(n) 最壞情況數字連續出現，stack 存滿
class Solution {
 public:
  int evalRPN(vector<string>& tokens) {
    stack<int> stk;
    for (const string& t : tokens) {
      if (t == "+" || t == "-" || t == "*" || t == "/") {
        int b = stk.top();  // 先彈出的是「右」運算元
        stk.pop();
        int a = stk.top();  // 後彈出的才是「左」運算元
        stk.pop();
        if (t == "+") stk.push(a + b);
        else if (t == "-") stk.push(a - b);
        else if (t == "*") stk.push(a * b);
        else stk.push(a / b);
      } else {
        stk.push(stoi(t));
      }
    }
    return stk.top();
  }
};
```

> [!warning] 先彈出的是「右」運算元
> stack 是 LIFO，後壓入的先彈出，所以第一個 `top()` 是運算式裡**靠右**的運算元。`["6","3","/"]` 該算 `6 / 3 = 2`，順序寫反就成了 `3 / 6 = 0`；加法乘法有交換律測不出來，減法除法一寫反就錯，這是本題最常見的 bug。

> [!note] 截斷除法與負數 token
> 題目要求除法「朝零截斷」，C++ 的整數 `/` 天生如此（`-7 / 3 = -2`），直接用即可；換成 Python 要小心 `//` 是向下取整（`-7 // 3 = -3`），得改用 `int(a / b)`。另外判斷運算子用**整字串比較**（`t == "-"`）天然安全——`"-11"` 不等於 `"-"`；若圖快用 `t[0] == '-'` 判斷，負數 token 就會被誤當成減號。

### 方法二：運算子查表（lambda map）— O(n)／O(n)

你的原解：把「運算子 → 二元函式」收進一張 `unordered_map`，主迴圈裡不含任何運算分支，`ops.contains(token)` 同時充當「是不是運算子」的判斷。擴充性最好——要支援 `%`、`^` 只需加一行表項；代價是 `function` 的型別抹除多一層間接呼叫、雜湊查找也比四次字串比較重一點，複雜度同為 O(n) 但常數稍大。

```cpp
// Time: O(n)
// Space: O(n)
class Solution {
 public:
  int evalRPN(vector<string>& tokens) {
    stack<int> nums;
    unordered_map<string, function<int(int, int)>> ops = {
      {"+", [](int a, int b) { return a + b; }},
      {"-", [](int a, int b) { return a - b; }},
      {"*", [](int a, int b) { return a * b; }},
      {"/", [](int a, int b) { return a / b; }}
    };

    for (const auto& token : tokens) {
      if (ops.contains(token)) {
        auto rhs = nums.top();  // 先彈出 = 右運算元
        nums.pop();
        auto lhs = nums.top();
        nums.pop();
        nums.push(ops[token](lhs, rhs));
      } else {
        nums.push(stoi(token));
      }
    }

    return nums.top();
  }
};
```

> [!tip] 原始碼的三處小修
> 一、`std::string` 改成 `string`，與 LeetCode 風格（`using namespace std;`）一致；二、range-for 用 `const auto&` 走訪，避免每個 token 複製一份 `string`；三、變數名從 `s`／`f` 改成 `rhs`／`lhs`，把「先彈出的是右運算元」直接寫進名字裡，順序 bug 一眼可查。

## Related Problems

[[0020-Valid-Parentheses]] — 同章 stack 基礎：同樣靠 LIFO「最近的先處理」語義掃描 token
[[0224-Basic-Calculator]] — 進階：中序表達式求值，需用 stack 處理括號與正負號
[[0227-Basic-Calculator-II]] — 進階：中序＋運算子優先級，RPN 正是把優先級預先攤平後的形式
[[0682-Baseball-Game]] — 同樣依 token 對 stack 壓入彈出的模擬題，本題的暖身款
