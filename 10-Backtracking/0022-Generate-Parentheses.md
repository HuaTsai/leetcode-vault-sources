---
leetcode-id: 22
difficulty: medium
tags:
  - string
  - dynamic-programming
  - backtracking
  - bracket-sequences
  - grind-169
  - neetcode-150
memo: 每一步只有放 `(` 或 `)` 兩種選擇，合法性靠不變式「任何前綴裡 `)` 的數量不超過 `(`」維持；守門條件是 `close < open` 而不是 `close < n`，寫錯就退化成枚舉全部 C(2n, n) 種排列、吐出 `())(` 這類前綴為負的字串
dg-publish: true
---

## Problem Description

Given `n` pairs of parentheses, write a function to generate all combinations of well-formed parentheses.

## Solution

核心觀念：字串合法的充要條件是**任何前綴裡 `)` 的數量都不超過 `(`，且最後兩者相等**。所以只要記著已放的 `open` 與 `close` 兩個計數，每一步的兩個選擇各有一個守門條件——放 `(` 要 `open < n`（還有存貨），放 `)` 要 `close < open`（有未配對的 `(` 可關）。條件擋在遞迴之前，走到底的每條路徑都必然合法，不需要事後驗證。

這題的形狀是 [[Backtracking-Templates]] 的**模板四（二元遞迴）**：枚舉的不是「下一個拿哪個元素」，而是「這一格填哪個字元」，所以一個 `bt` 裡出現兩次遞迴呼叫、外面不該再有 `for`。

```txt
n = 2，狀態 (open, close)，守門條件寫在邊上

(0,0) ""
├─ '('  open<2 ✓ ── (1,0) "("
│                    ├─ '('  open<2 ✓ ── (2,0) "(("
│                    │                    └─ ')'  close<open ✓ ── (2,1) "(()"
│                    │                                             └─ ')' ✓ ── (2,2) "(())" ★
│                    └─ ')'  close<open ✓ ── (1,1) "()"
│                                             ├─ '('  open<2 ✓ ── (2,1) "()("
│                                             │                    └─ ')' ✓ ── (2,2) "()()" ★
│                                             └─ ')'  close<open ✗ 剪掉（"())" 前綴已非法）
└─ ')'  close<open ✗ 剪掉（")" 一開頭就非法）
```

> [!important] 兩個條件是不對稱的
> `open < n` 比的是**存貨**，`close < open` 比的是**另一個計數**。前者只管「還有沒有左括號能放」，後者才是維持合法性的那道不變式。把後者也寫成跟 `n` 比（`close < n`）就等於完全不剪枝——見方法一底下的實測。

答案數是卡塔蘭數 $C_n = \frac{1}{n+1}\binom{2n}{n} \approx \frac{4^n}{n^{1.5}\sqrt{\pi}}$，每個答案長度 `2n`、複製要 `O(n)`，所以以下三個方法的時間都是 `O(4^n/√n)`，差別只在常數與額外空間。

### 方法一：open／close 雙計數回溯 — O(4^n/√n)／O(n)

最正規的寫法，面試講得最快、也最不會寫錯。

```cpp
// Time: O(4^n/sqrt(n))  卡塔蘭數個答案，每個複製 O(n)
// Space: O(n)           遞迴深度 2n + cur，不計輸出
class Solution {
 public:
  vector<string> generateParenthesis(int n) {
    this->n = n;
    bt(0, 0);
    return ans;
  }

 private:
  void bt(int open, int close) {
    if ((int)cur.size() == 2 * n) {
      ans.push_back(cur);
      return;
    }
    if (open < n) {  // 還有 '(' 可用
      cur.push_back('(');
      bt(open + 1, close);
      cur.pop_back();
    }
    if (close < open) {  // 有未配對的 '(' 才能關
      cur.push_back(')');
      bt(open, close + 1);
      cur.pop_back();
    }
  }

  int n;
  string cur;
  vector<string> ans;
};
```

> [!warning] 是 `close < open`，不是 `close < n`
> 兩個條件都能保證「`)` 總共只放 n 個」，但只有前者管住**前綴**。寫成 `close < n` 之後每一格都能自由填兩種字元，等於枚舉 $\binom{2n}{n}$ 種排列全部收錄。實測 `n = 2`：
>
> ```txt
> 正解（2 筆）      (())  ()()
> close < n（6 筆） (())  ()()  ())(  )(()  )()(  ))((
> ```
>
> 多出來的四筆前綴都曾經為負。這種錯不會 crash、小測資看起來還「多幾筆而已」，只有把非法字串印出來才抓得到。

### 方法二：`(` 用完後一次補齊剩下的 `)` — O(4^n/√n)／O(n)

當 `open == n` 時遞迴其實已經沒有選擇了：剩下的位置**必然**全是 `)`，數量固定是 `n - close`。與其一層一層放下去，不如直接一次補完，把每條路徑尾端那串線性遞迴鏈整個砍掉。

```cpp
// Time: O(4^n/sqrt(n))
// Space: O(n)
class Solution {
 public:
  vector<string> generateParenthesis(int n) {
    this->n = n;
    bt(0, 0);
    return ans;
  }

 private:
  void bt(int open, int close) {
    if (open == n) {  // '(' 用完 -> 剩下必然是 n - close 個 ')'
      size_t len = cur.size();
      cur.append(n - close, ')');
      ans.push_back(cur);
      cur.resize(len);  // 跟 pop_back 一樣是撤銷，只是一次撤好幾個
      return;
    }
    cur.push_back('(');
    bt(open + 1, close);
    cur.pop_back();

    if (open > close) {
      cur.push_back(')');
      bt(open, close + 1);
      cur.pop_back();
    }
  }

  int n;
  string cur;
  vector<string> ans;
};
```

> [!note] `open == n` 這個 base case 同時吃掉了原本的兩種收尾
> 不必再分「`close` 也滿了」和「還沒滿」兩條路：`close == n` 時 `append(0, ')')` 是空操作，`push_back` 到的就是 `cur` 本身。多寫一個 `if (close == n)` 分支只是重複。

> [!warning] 遞迴節點少一半，時間卻幾乎沒省
> 這是本題最值得記的一件事。n = 12（208,012 筆答案）用 cachegrind 數指令：
>
> | 版本 | 遞迴節點數 | 只跑遞迴不建字串 | 完整（含輸出） |
> | --- | --- | --- | --- |
> | 方法一 標準 | 1,033,411 | 65.2 M | 145.9 M |
> | 方法二 寫成 `cur + string(n - close, ')')` | 498,523（-52%） | 28.5 M（-56%） | **146.5 M（+0.4%）** |
> | 方法二 就地 `append` / `resize`（上面的版本） | 498,523（-52%） | 28.5 M（-56%） | 132.2 M（-9.4%） |
>
> 節點數確實砍了一半，隔離出來只跑遞迴也確實快 56%。但**完整版本的時間被「配置 $C_n$ 個長度 2n 的字串」主導**，遞迴本身不到一半（65.2 M / 145.9 M），所以 -52% 的節點最多只能換到 -9% 的總指令。
>
> 更諷刺的是 `cur + string(n - close, ')')` 這個寫法：它每次都生兩個暫時字串（`string(n - close, ')')` 一個、`operator+` 的結果一個），多花的配置成本剛好把省下的遞迴全部吃回去，總指令反而比標準版還多 0.4%。改成就地 `append` + `resize`（零暫時物件）才真的拿得到那 9%。

### 方法三：卡塔蘭分解 DP — O(4^n/√n)／O(4^n/√n)

換個角度：**每個合法字串都能唯一寫成 `( A ) B`**——`A` 是第一個 `(` 所配對的 `)` 之間那段，`B` 是它後面那段，兩段各自合法。枚舉 `A` 用掉幾對括號就得到遞推式，這正是卡塔蘭數 $C_k = \sum_{i=0}^{k-1} C_i C_{k-1-i}$ 的組合意義。

```txt
k = 3，切點 i = A 裡的對數

i=0  ( )  BB     -> "()" + f[2] 的每一個
i=1  ( A ) B     -> "(" + f[1] + ")" + f[1]
i=2  ( AA )      -> "(" + f[2] + ")" + ""
```

```cpp
// Time: O(4^n/sqrt(n))  同階，但字串串接的常數比回溯大（實測 n=14 慢約 2 倍）
// Space: O(4^n/sqrt(n)) 所有中間層都要留著，額外約佔最終答案的 35%
class Solution {
 public:
  vector<string> generateParenthesis(int n) {
    vector<vector<string>> f(n + 1);
    f[0] = {""};
    for (int k = 1; k <= n; ++k) {
      for (int i = 0; i < k; ++i) {  // 首個 '(' 的配對括號裡放 i 對
        for (const string& in : f[i]) {
          for (const string& out : f[k - 1 - i]) {
            f[k].push_back("(" + in + ")" + out);
          }
        }
      }
    }
    return f[n];
  }
};
```

> [!tip] 分解唯一，所以不重不漏
> 「第一個 `(` 的配對位置」對每個合法字串是**唯一確定**的，所以不同的 `i` 產生的字串必然不同（`A` 的長度不同），不需要任何去重。這跟回溯用剪枝保證不重是兩種不同的手法——一個靠結構唯一性，一個靠決策不重複。

> [!note] 什麼時候值得用 DP 版
> 本題只要列舉，回溯完勝（空間 `O(n)` vs `O(4^n/√n)`、常數也小一半）。DP 版的價值在**題目改問「有幾種」**——那時整個 `f` 退化成一維整數陣列，時間掉到 `O(n^2)`、空間 `O(n)`，回溯反而完全做不動。見 [[Knapsack-and-Classic-DP]] 的同一條分界線。

## Related Problems

- [[0020-Valid-Parentheses]] — 同一個「前綴 `)` 不超過 `(`」不變式的驗證版；本題是拿它當剪枝條件去生成
- [[0078-Subsets]] — 同樣是模板四的二元遞迴骨架，差別在兩個分支從「選／不選」換成「放 `(` ／放 `)`」，且各自帶守門條件
- [[0090-Subsets-II]] — 對照組：那題兩套骨架（有 `for` ／二元遞迴）都能寫，本題只有二元遞迴這一種形狀
- [[0039-Combination-Sum]] — 同樣是「剪枝擋在遞迴之前」，那題比的是剩餘 target，這題比的是 open／close
- [[Backtracking-Templates]] — 模板四的通則，以及「一個 `bt` 兩次遞迴就不該再有 `for`」的判斷法
- [[Knapsack-and-Classic-DP]] — 方法三的卡塔蘭遞推是區間 DP 的入門形狀；題目從「列出所有」改問「有幾種」時的換軌點
- [[Micro-Optimization-Myths]] — 方法二的教訓同源：省下的操作數不等於省下的時間，得先確認瓶頸在哪
- [[STL-Pitfalls]] — `cur + string(...)` 生暫時物件、`append` / `resize` 就地操作的成本差
