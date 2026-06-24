---
leetcode-id: 155
difficulty: medium
tags:
  - stack
  - design
  - grind-169
  - neetcode-150
memo: getMin 要 O(1) 就不能查詢時才掃；訣竅是另開一個 minstk 與主 stack 同步 push／pop、每層存「到目前為止的最小值」，lazy 版只記錄打破紀錄者時務必用小於等於比較，否則重複的最小值會漏記
dg-publish: true
---

## Problem Description

Design a stack that supports push, pop, top, and retrieving the minimum element in constant time.

Implement the `MinStack` class:

- `MinStack()` initializes the stack object.
- `void push(int value)` pushes the element `value` onto the stack.
- `void pop()` removes the element on the top of the stack.
- `int top()` gets the top element of the stack.
- `int getMin()` retrieves the minimum element in the stack.

You must implement a solution with `O(1)` time complexity for each function.

## Solution

核心觀念：`getMin` 要 O(1)，代表**不能在查詢的當下才去掃整個 stack**，必須在 push／pop 的當下就把「目前為止的最小值」維護好，讓它隨取隨用。關鍵性質是最小值會**隨 stack 單調回溯**——pop 掉一個元素後，最小值必定回到「該元素被壓入之前」的值，這恰好能用另一個與主 stack 同步升降的 stack 來記錄。

```txt
push 順序: -2, 0, -3
stk    : [-2,  0, -3]
minstk : [-2, -2, -3]   每層存「到這一層為止的最小值」
                    ↑ top 永遠是當前最小值 = -3
pop 一次後:
stk    : [-2,  0]
minstk : [-2, -2]        最小值自動回到 -2，不需重算
```

> [!important] 用空間換查詢時間
> 單一 stack 求最小值只能 O(n) 掃描。本題的解法統一是「**多存一份輔助資訊**」，把求最小值的成本從查詢時攤提到 push 時，四個操作就全是 O(1)。

### 方法一：雙 stack，每層存 prefix min — O(1)／O(n)（推薦）

`minstk` 與主 stack 一一對應，第 `i` 層存「stk 前 `i` 個元素的最小值」，也就是 `min(舊棧頂, 新值)`。兩個 stack 永遠同步 push／pop，所以 `minstk.top()` 直接就是當前最小值。邏輯最直白、不需任何特例。

```cpp
// Time: push／pop／top／getMin 皆 O(1)
// Space: O(n)  minstk 與主 stack 等長
class MinStack {
 public:
  MinStack() {}

  void push(int value) {
    stk.push(value);
    if (minstk.empty()) {
      minstk.push(value);
    } else {
      minstk.push(min(minstk.top(), value));
    }
  }

  void pop() {
    stk.pop();
    minstk.pop();
  }

  int top() { return stk.top(); }

  int getMin() { return minstk.top(); }

 private:
  stack<int> stk;
  stack<int> minstk;
};
```

### 方法二：lazy min stack，只記錄打破紀錄者 — O(1)／O(n)

方法一不論值多大都往 `minstk` 塞一份，其實只有「**追平或刷新最小值**」的元素才值得記。lazy 版只在 `value <= minstk.top()` 時才 push；pop 時也只有當被彈出的值正好等於 `minstk.top()`（代表它就是當前最小值）才同步彈掉。最壞情況（嚴格遞減）仍與主 stack 等長，但一般輸入會省下可觀空間。

```cpp
// Time: push／pop／top／getMin 皆 O(1)
// Space: O(n)  最壞仍與主 stack 等長，一般情況更省
class MinStack {
 public:
  MinStack() {}

  void push(int value) {
    if (minstk.empty() || value <= minstk.top()) minstk.push(value);
    stk.push(value);
  }

  void pop() {
    if (stk.top() == minstk.top()) minstk.pop();
    stk.pop();
  }

  int top() { return stk.top(); }

  int getMin() { return minstk.top(); }

 private:
  stack<int> stk;
  stack<int> minstk;
};
```

> [!warning] 重複最小值必須用 `<=` 搭配 `==`
> push 的判斷若寫成 `value < minstk.top()`，遇到 `[0, 1, 0]` 這種有重複最小值的序列時，第二個 `0` 不會進 `minstk`；之後 pop 掉它時 `stk.top() == minstk.top()` 又成立、把 `minstk` 裡僅有的那個 `0` 也彈掉，導致底層還在的 `0` 找不到最小值紀錄。push 用 `<=`、pop 用 `==`，才能讓相等的最小值各自獨立記一筆。

## Related Problems

[[0020-Valid-Parentheses]] — stack 最基礎的操作題，本題的暖身
[[0716-Max-Stack]] — 直接變形：把 getMin 換成 getMax 並多一個 popMax，難度升級
[[0232-Implement-Queue-using-Stacks]] — 同為「用既有結構設計新資料結構」的 design 題型
[[0739-Daily-Temperatures]] — monotonic stack，與本題「維護額外單調資訊以換取 O(1) 查詢」同源
