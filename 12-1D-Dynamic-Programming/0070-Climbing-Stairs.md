---
leetcode-id: 70
difficulty: easy
tags:
  - dynamic-programming
  - grind-169
  - neetcode-150
memo: 最後一步只能跨 1 或 2 階，f(n) = f(n-1) + f(n-2) 就是費氏數列，滾動兩個變數即可；起點取 f(0) = f(1) = 1 可免特判
dg-publish: true
---

## Problem Description

You are climbing a staircase. It takes `n` steps to reach the top.

Each time you can either climb `1` or `2` steps. In how many distinct ways can you climb to the top?

## Solution

站在第 n 階回頭看，最後一步只有兩種可能：從第 n−1 階跨 1 階上來，或從第 n−2 階跨 2 階上來。兩種情況互斥又涵蓋全部，所以走法數直接相加：`f(n) = f(n-1) + f(n-2)`，就是位移一格的費氏數列。

```txt
n    : 0  1  2  3  4  5
f(n) : 1  1  2  3  5  8
                  ↑
                  f(4) = f(3) + f(2) = 3 + 2
                  最後一步跨 1 階（從 3 上來）或跨 2 階（從 2 上來）
```

> [!important]
> 每一格只依賴前兩格，不需要整個 dp 陣列，留兩個變數往前滾就好，空間從 O(n) 降到 O(1)。

### 方法一：滾動變數 — O(n)／O(1)

起點取 `f(0) = 1`、`f(1) = 1`（0 階有一種走法：不動），迴圈從 2 開始，n = 1 和 n = 2 都自然落在迴圈裡，不必特判。

```cpp
// Time: O(n)
// Space: O(1)
class Solution {
 public:
  int climbStairs(int n) {
    int prev = 1;
    int cur = 1;
    for (int i = 2; i <= n; ++i) {
      prev = exchange(cur, prev + cur);
    }
    return cur;
  }
};
```

> [!tip]
> `exchange(cur, prev + cur)` 把 `cur` 設成新值、回傳舊值，一行講完「`cur` 變成和，`prev` 接住舊的 `cur`」。等價寫法是 `prev += cur; swap(prev, cur);`：`prev` 先暫時變成新的 `cur` 再換回來，結果相同，但讀的人要在腦中多跑一步。

> [!note]
> 原本的寫法是特判 `n == 1`、`n == 2` 後從 `prev = 1, cur = 2`、`i = 3` 起跳，也正確。把起點往前推一格到 `f(0)`，兩個特判就都消失了。

> [!warning]
> 回傳型別 `int` 是剛好夠用：f(45) = 1836311903 < `INT_MAX` = 2147483647，f(46) = 2971215073 就溢位了。題目若把 n 放寬，要改 `long long` 或取模。

### 方法二：矩陣快速冪 — O(log n)／O(1)

追問「n 到 10¹⁸ 怎麼辦」時的答案。把遞推寫成矩陣乘法，走 n 步就是同一個矩陣自乘 n 次，而冪次可以用快速冪在 O(log n) 內算完（骨架同 [[Modular-Arithmetic]] 的 `powmod`，只是把數的乘法換成矩陣乘）。

```txt
| f(n+1) |   | 1 1 |   | f(n)   |            | 1 1 |^n   | f(n)   f(n-1) |
| f(n)   | = | 1 0 | · | f(n-1) |     →      | 1 0 |   = | f(n-1) f(n-2) |

n = 1 時右邊是 | f(1) f(0) ; f(0) 0 | = | 1 1 ; 1 0 |，正好是矩陣本身。
答案就是 n 次方後的左上角 [0][0]。
```

```cpp
// Time: O(log n)
// Space: O(1)
class Solution {
 public:
  int climbStairs(int n) {
    Mat res{{{1, 0}, {0, 1}}};
    Mat base{{{1, 1}, {1, 0}}};
    for (; n > 0; n >>= 1) {
      if (n & 1) {
        res = mul(res, base);
      }
      base = mul(base, base);
    }
    return res[0][0];
  }

 private:
  using Mat = array<array<long long, 2>, 2>;

  static Mat mul(const Mat& a, const Mat& b) {
    Mat c{};
    for (int i = 0; i < 2; ++i) {
      for (int j = 0; j < 2; ++j) {
        for (int k = 0; k < 2; ++k) {
          c[i][j] += a[i][k] * b[k][j];
        }
      }
    }
    return c;
  }
};
```

> [!warning]
> 矩陣元素要用 `long long`，即使答案本身塞得進 `int`。快速冪每輪都會把 `base` 平方，最後一輪平方出來的值沒人用，卻照樣會算：n ≥ 32 時 `base` 會被平方到 64 次方，裡面是 f(64) 等級的數（約 10¹³）。換成 `int` 用 UBSan 跑會報 `3524578 * 3524578` signed overflow，雖然 1..45 的答案碰巧都還對，但已經是 UB。

> [!note]
> 這題 n ≤ 45，方法一最多跑 44 圈，矩陣快速冪不會比較快，是為了接住放寬 n 的追問。真的放寬到 10¹⁸ 時答案必然要取模，`mul` 裡每次乘加後 `% m` 即可。

> [!note]
> 還有閉式解 Binet 公式：`f(n) = round(φ^(n+1) / √5)`，φ = (1 + √5) / 2。用 `double` 實測 n = 1..45 全對，但到 n = 70 就差 1（浮點精度不夠），所以只能當冷知識，不能當放寬 n 的解。

## Related Problems

- [[0509-Fibonacci-Number]] — 同一條遞推，只差起點位移一格
- [[1137-N-th-Tribonacci-Number]] — 依賴前三格，滾動變數從兩個變三個，矩陣變 3×3
- [[0746-Min-Cost-Climbing-Stairs]] — 同樣從前一、二階轉移，但把「相加計數」換成「取 min 加成本」
- [[0198-House-Robber]] — 同為只看前兩格的 1-D DP，轉移改成「偷／不偷」取 max
- [[0091-Decode-Ways]] — 一次吃 1 或 2 個字元的計數 DP，是加了合法性條件的爬樓梯
- [[Modular-Arithmetic]] — 快速冪骨架，方法二把數的乘法換成矩陣乘
