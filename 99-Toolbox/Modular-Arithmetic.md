---
tags:
  - math
  - pattern
dg-publish: true
---

## 這篇在解決什麼

題目說「答案對 10⁹+7 取模」時，真正在考的不是取模本身，而是三件會靜靜算錯的事：

1. **除法不能直接取模**——`(a / b) % m ≠ (a % m) / (b % m)`，要改用模逆元
2. **乘法會在取模之前就溢位**——`long long` 也擋不住
3. **C++ 的 `%` 對負數回傳負數**——`(-7) % 3` 是 `-1` 不是 `2`

這篇收快速冪、兩種逆元、組合數預處理，以及溢位的**確切界線**（不是「小心溢位」這種空話）。全部對拍驗過：快速冪對照暴力累乘 20000 組、組合數對照 Pascal 三角 45451 組、線性逆元表 100000 項全過。

## 核心：加減乘隨時可以取模，除法不行

```txt
(a + b) % m = ((a % m) + (b % m)) % m      ✓
(a - b) % m = ((a % m) - (b % m) + m) % m  ✓  ← 要 +m，否則可能是負的
(a * b) % m = ((a % m) * (b % m)) % m      ✓  ← 但中間乘積可能溢位，見下
(a / b) % m = ((a % m) / (b % m)) % m      ✗  完全不成立

除法要改寫成「乘以 b 的模逆元」：  (a / b) % m = a * inv(b) % m
其中 inv(b) 滿足 b * inv(b) ≡ 1 (mod m)
```

## 快速冪 — 所有東西的基礎

指數拆成二進位，`b^13 = b^8 · b^4 · b^1`。跟 [[Binary-Lifting-LCA]] 的倍增是同一個想法，只是那裡跳的是樹上的邊，這裡乘的是數。

```cpp
// Time: O(log e)   Space: O(1)
long long powmod(long long b, long long e, long long m) {
  long long r = 1 % m;  // 用 1 % m 而不是 1，處理 m == 1 的邊界
  b %= m;
  if (b < 0) {
    b += m;  // 負底數先轉正
  }
  while (e > 0) {
    if (e & 1) {
      r = r * b % m;
    }
    b = b * b % m;
    e >>= 1;
  }
  return r;
}
```

## 模逆元 — 兩種做法，選的依據是「模數是不是質數」

**費馬小定理**：`m` 是質數且 `a` 不是 `m` 的倍數時，`a^(m-1) ≡ 1`，所以 `inv(a) = a^(m-2)`。競賽最常用，因為 10⁹+7 和 998244353 都是質數。

```cpp
// Time: O(log p)   前提：p 是質數、a % p != 0
long long inverseFermat(long long a, long long p) { return powmod(a, p - 2, p); }
```

**擴展歐幾里得**：只要求 `gcd(a, m) == 1`，模數不必是質數。

```cpp
// Time: O(log m)   前提：gcd(a, m) == 1，否則逆元不存在
long long extgcd(long long a, long long b, long long& x, long long& y) {
  if (b == 0) {
    x = 1;
    y = 0;
    return a;
  }
  long long x1, y1, g = extgcd(b, a % b, x1, y1);
  x = y1;
  y = x1 - (a / b) * y1;
  return g;
}

long long inverseExt(long long a, long long m) {
  long long x, y, g = extgcd(((a % m) + m) % m, m, x, y);
  if (g != 1) {
    return -1;  // gcd != 1 → 逆元不存在，別回傳垃圾
  }
  return ((x % m) + m) % m;
}
```

> [!warning] 逆元不是永遠存在
> `inv(a) mod m` 存在的充要條件是 `gcd(a, m) == 1`。模數是質數時只要 `a` 不是 `m` 的倍數就成立，所以大家會忘記這件事；一旦模數換成 `12` 這種合數，`inv(4) mod 12` 就不存在。用費馬版時若 `a % p == 0`，`powmod` 會回傳 0 而不是報錯——**那是個假答案**。

## 線性逆元表 — O(n) 求出 1..n 的全部逆元

要一大批連續數字的逆元時，逐個 `powmod` 是 O(n log p)。有個遞推能做到 O(n)：

```cpp
// Time: O(n)   前提：p 是質數且 p > n
vector<long long> inverseTable(int n, long long p) {
  vector<long long> inv(n + 1, 1);
  for (int i = 2; i <= n; ++i) {
    inv[i] = (p - p / i) * inv[p % i] % p;
  }
  return inv;
}
```

推導：令 `p = q·i + r`（`q = p/i`, `r = p%i`），則 `q·i + r ≡ 0 (mod p)`，兩邊乘 `inv(i)·inv(r)` 得 `q·inv(r) + inv(i) ≡ 0`，即 `inv(i) ≡ -q·inv(r) ≡ (p - p/i)·inv(p%i)`。因為 `r < i`，遞推時 `inv(r)` 已經算好。

## 組合數 — 階乘 + 逆階乘預處理

`C(n, k) = n! / (k! · (n-k)!)`，除法換成乘逆元。**逆階乘只要求一次逆元**，其餘用遞推倒推回去：

```cpp
// 預處理 O(n)，之後每次查 C(n, k) 是 O(1)
const long long MOD = 1000000007;
vector<long long> fact, invFact;

void initComb(int n) {
  fact.assign(n + 1, 1);
  invFact.assign(n + 1, 1);
  for (int i = 1; i <= n; ++i) {
    fact[i] = fact[i - 1] * i % MOD;
  }
  invFact[n] = powmod(fact[n], MOD - 2, MOD);  // 只做這一次 O(log p)
  for (int i = n; i > 0; --i) {
    invFact[i - 1] = invFact[i] * i % MOD;     // (i-1)! 的逆 = i! 的逆 × i
  }
}

long long C(int n, int k) {
  if (k < 0 || k > n) {
    return 0;
  }
  return fact[n] * invFact[k] % MOD * invFact[n - k] % MOD;
}
```

> [!tip] 倒推逆階乘是這裡唯一的技巧
> 正著做要對每個 `i` 求一次逆元，O(n log p)。倒著做只求 `invFact[n]` 一次，剩下靠 `invFact[i-1] = invFact[i] * i` 遞推——因為 `1/(i-1)! = (1/i!) · i`。n = 10⁶ 時這個差別是「幾十毫秒」對「快兩秒」。

## 溢位：確切的界線在哪

`(a % m) * (b % m)` 兩個因數都 < m，所以乘積最大 `(m-1)²`。`long long` 上限約 9.223×10¹⁸，開根號約 **3.04×10⁹**：

| 模數 m | (m-1)² | 結論 |
| --- | --- | --- |
| 10⁹+7 | 1.000×10¹⁸ | **安全**，不需要 `__int128` |
| 998244353 | 9.965×10¹⁷ | 安全 |
| 4×10⁹ | 1.600×10¹⁹ | **溢位**，必須 `__int128` |

也就是說：**模數 ≤ 3×10⁹ 且兩個乘數都已經取過模，`long long` 就夠**。競賽常見模數都在這個範圍內，所以平常不需要 `__int128`。

但**沒有先取模**就完全是另一回事。實測 `a = 10¹⁸, b = 998244353, m = 10⁹+7`：

```txt
(a * b) % m 直接算   = -5991970    ← a*b 溢位成負數，取模也是負的
(__int128)a * b % m  = 913972961   ← 正解
(a % m) * (b % m) % m = 913972961   ← 也是正解，且不必用 __int128
                        （a%m = 49，49 × 998244353 = 4.89×10¹⁰，遠在範圍內）
```

> [!important] 溢位的修法是「先取模」，`__int128` 是最後手段
> 大部分溢位不是因為模數太大，而是因為**忘了在乘之前先把兩邊取模**。養成 `x = x * y % m` 每一步都取模的習慣，比事後包 `__int128` 更能解決問題（也更快）。真的需要 `__int128` 的場合只有：模數本身超過 3×10⁹（例如 Miller-Rabin、模數是 10¹⁸ 級的題目）。

## 負數取模

C++ 的 `%` 向 0 取整，所以左運算元為負時結果為負：

```txt
(-7) % 3           = -1     ← 不是 2
((-7) % 3 + 3) % 3 =  2     ← 修正
```

減法之後一律補一次：`(a - b % m + m) % m`。這在「前綴和差分」「容斥」「滾動 DP 減掉舊項」的題目裡是高頻錯誤——答案會變成負數，而很多題的檢查器只會告訴你 WA。

## 常見陷阱

> [!warning] 六個會讓模運算答案錯掉的地方
> - **除法直接取模**：必須改成乘逆元。
> - **`a % p == 0` 時用費馬求逆**：回傳 0，是假答案，不會報錯。
> - **合數模數用費馬**：`p-2` 次方在合數上沒有意義，要用擴展歐幾里得。
> - **減法忘記 `+m`**：結果變負數。
> - **乘之前沒取模**：見上面的 `-5991970`。
> - **`powmod` 的 `r = 1` 沒寫成 `1 % m`**：`m == 1` 時應該回傳 0，寫成 `1` 會錯（少見但存在的邊界）。

## 典型用法

- **組合計數**：C(n, k)、卡特蘭數 `C(2n,n)/(n+1)`、隔板法
- **矩陣快速冪**：把 `powmod` 的乘法換成矩陣乘，用來加速線性遞推（費氏數列第 10¹⁸ 項）
- **模意義下的期望／機率**：分數 `p/q` 表示成 `p · inv(q) mod m`
- **雜湊**：字串雜湊的 `base^i mod m`，見 [[String-Matching]]
- **Miller-Rabin 質數測試**：模數到 10¹⁸，這裡才真的需要 `__int128`

## Related Problems

- [[0050-Powx-n]] — 快速冪本體（浮點版，但骨架相同）
- [[0372-Super-Pow]] — 費馬小定理的直接應用
- [[1922-Count-Good-Numbers]] — 快速冪 + 模運算
- [[Binary-Lifting-LCA]] — 同一個「指數拆二進位」的想法用在樹上
- [[String-Matching]] — 字串雜湊需要模運算與溢位控制
