---
tags:
  - string
  - pattern
dg-publish: true
---

## 這篇在解決什麼

「在長度 n 的文字裡找長度 m 的樣式」，暴力是 O(nm)。三個標準工具都能做到 O(n + m)，但它們回答的問題不只是「有沒有出現」：

- **KMP** 的副產品 `pi[]` 回答「每個前綴的最長 border 是多長」——週期、循環節、最短補全都靠它
- **Z-function** 的 `z[]` 回答「從每個位置開始，和整個字串的最長共同前綴多長」
- **字串雜湊**回答「任兩個子字串相不相等」，O(1) 一次，代價是有碰撞機率

這篇三個都收，並實測雜湊被打爆的確切條件。對拍暴力驗過：KMP 與 Z 的搜尋結果、`pi[]` 與 `z[]` 的定義式、雜湊比對，各 4000 組隨機字串全過。

## KMP — `pi[i]` 是「s[0..i] 的最長真 border」

**border** = 既是前綴又是後綴的字串（不含自己）。

```txt
s  =  a  b  a  c  a  b  a  b
i     0  1  2  3  4  5  6  7
pi    0  0  1  0  1  2  3  2

pi[6] = 3：s[0..6] = "abacaba" 的最長 border 是 "aba"
           a b a c a b a
           └──┬──┘ └──┬──┘
             "aba"   "aba"      前綴和後綴重疊也沒關係

失配時就是靠這個往回跳：已經匹配了 j 個字元卻在第 j+1 個失配，
不必從頭再來，跳到 pi[j-1] 繼續即可——因為那 pi[j-1] 個字元一定也匹配。
```

```cpp
// Time: O(n)   Space: O(n)
vector<int> prefixFunction(const string& s) {
  int n = s.size();
  vector<int> pi(n, 0);
  for (int i = 1; i < n; ++i) {
    int j = pi[i - 1];
    while (j > 0 && s[i] != s[j]) {
      j = pi[j - 1];  // 失配就退到次長的 border 再試
    }
    if (s[i] == s[j]) {
      ++j;
    }
    pi[i] = j;
  }
  return pi;
}

// 找出 pat 在 text 中所有出現位置（0-indexed 起點）
vector<int> kmpSearch(const string& text, const string& pat) {
  vector<int> res;
  if (pat.empty()) {
    return res;
  }
  string s = pat + '\x01' + text;  // 分隔符必須是兩邊都不會出現的字元
  auto pi = prefixFunction(s);
  int m = pat.size();
  for (int i = m + 1; i < (int)s.size(); ++i) {
    if (pi[i] == m) {
      res.push_back(i - 2 * m);
    }
  }
  return res;
}
```

> [!important] 那個分隔符不能省
> 沒有分隔符的話，`pi` 值可能超過 `m`（樣式和文字接起來產生了更長的 border），就會漏報或誤報。用 `'\x01'`、`'#'`、`'$'` 都行，但**必須確定它不在輸入的字元集裡**——題目說「只有小寫字母」時 `#` 很安全，說「任意 ASCII」時就得挑別的（或改用不拼接的雙指標版 KMP）。

> [!tip] `pi` 最有價值的用途不是搜尋，是週期
> 對長度 n 的字串 `s`，令 `k = n - pi[n-1]`：
>
> - `k` 是 `s` 的**最小週期**（若 `n % k == 0`，則 `s` 由 `n/k` 個長度 k 的區塊重複而成）
> - 「最少要在後面補幾個字元才能讓 s 變成某個字串的重複」= `k - n % k`（整除時為 0）
>
> CSES / LeetCode 上一大票「重複子字串」題目就是這兩行。

## Z-function — `z[i]` 是「s 和 s[i..] 的最長共同前綴」

```txt
s  =  a  a  b  x  a  a  y  a  a
i     0  1  2  3  4  5  6  7  8
z     9  1  0  0  2  1  0  2  1
      ↑
   z[0] 定義成 n（或不使用）

z[4] = 2：從位置 4 開始的 "aay.." 和開頭的 "aab.." 共同前綴是 "aa"
```

```cpp
// Time: O(n)   Space: O(n)
vector<int> zFunction(const string& s) {
  int n = s.size();
  vector<int> z(n, 0);
  if (n) {
    z[0] = n;
  }
  for (int i = 1, l = 0, r = 0; i < n; ++i) {
    if (i < r) {
      z[i] = min(r - i, z[i - l]);  // 借用先前算過的區段，免掉重複比較
    }
    while (i + z[i] < n && s[z[i]] == s[i + z[i]]) {
      ++z[i];
    }
    if (i + z[i] > r) {
      l = i;
      r = i + z[i];  // 維護「最右邊的匹配區段」
    }
  }
  return z;
}
```

搜尋一樣用拼接：`z[i] >= m` 的位置就是一次出現。

> [!note] KMP 和 Z 大部分場合可以互換，挑順手的那個
> 兩者都是 O(n)、都靠「拼接 + 分隔符」做搜尋。差別在副產品的語意不同：**要 border／週期用 KMP**，**要「和開頭比對多長」用 Z**（例如「把字串切成兩半，前半是不是後半的前綴」這種題，Z 一行就答完）。實測兩者搜尋結果在 4000 組隨機測資上完全一致。

## 字串雜湊 — O(1) 比較任兩個子字串

把字串看成 base 進位的數：`h(s) = s[0]·B^(n-1) + s[1]·B^(n-2) + … + s[n-1]`，取模避免溢位。前綴雜湊算好之後，任意子字串的雜湊是 O(1)：

```cpp
// build: O(n)   get: O(1)
// 雙模：兩個模數同時碰撞的機率才是問題，見下面實測
struct StringHash {
  static const long long M1 = 1000000007, M2 = 998244353, B1 = 131, B2 = 137;
  vector<long long> h1, h2, p1, p2;

  explicit StringHash(const string& s) {
    int n = s.size();
    h1.assign(n + 1, 0);
    h2.assign(n + 1, 0);
    p1.assign(n + 1, 1);
    p2.assign(n + 1, 1);
    for (int i = 0; i < n; ++i) {
      h1[i + 1] = (h1[i] * B1 + s[i]) % M1;
      h2[i + 1] = (h2[i] * B2 + s[i]) % M2;
      p1[i + 1] = p1[i] * B1 % M1;
      p2[i + 1] = p2[i] * B2 % M2;
    }
  }

  pair<long long, long long> get(int l, int r) {  // 閉區間 [l, r]
    long long a = ((h1[r + 1] - h1[l] * p1[r - l + 1]) % M1 + M1) % M1;
    long long b = ((h2[r + 1] - h2[l] * p2[r - l + 1]) % M2 + M2) % M2;
    return {a, b};  // 兩個一起比才算相等
  }
};
```

減法之後那個 `+ M1) % M1` 是必要的——C++ 的 `%` 對負數回傳負數，見 [[Modular-Arithmetic]]。

### 雜湊會被打爆，而且不是理論上而已

**mod 2⁶⁴ 自然溢位（`unsigned long long` 不取模）是最危險的偷懶做法。** Thue-Morse 字串能構造出必然碰撞的反例，實測：

| 字串長度 | base 131 | base 137 |
| --- | --- | --- |
| 16–512 | 沒碰撞 | 沒碰撞 |
| **1024** | **碰撞** | **碰撞** |
| **2048** | **碰撞** | **碰撞** |
| **4096** | **碰撞** | **碰撞** |

> [!warning] Thue-Morse 攻擊與 base 無關——換 base 救不了 mod 2⁶⁴
> 上表兩個不同的 base **同時**碰撞。這個構造利用的是 2 的冪次模數的結構，不是特定 base 的弱點，所以「我換一個奇怪的 base 就好」是無效的防禦。Codeforces 上的 hack 就是靠這個。**要嘛用質數模數，要嘛別用雜湊。**

**模數太小則會敗給生日悖論。** 20000 個長度 10 的隨機字串：

| 模數 | 不同雜湊值 | 碰撞組數 |
| --- | --- | --- |
| 10⁶+3 | 19814 | **186** |
| 10⁹+7 | 20000 | 0 |
| 2⁶¹−1 | 20000 | 0 |

同一組 Thue-Morse 串（長度 4096）對 mod 10⁹+7 **沒有**碰撞——質數模數擋掉了這個構造。

> [!tip] 實務建議
> - **雙模** `(10⁹+7, 998244353)`，或**單模 2⁶¹−1** 配 `__int128` 乘法
> - base 取一個大於字元集大小的值（131、137、31 都常見）；比賽中怕被針對就**隨機挑 base**
> - 絕對不要 mod 2⁶⁴ 自然溢位
> - 有確定性解法（KMP／Z）時優先用它們——雜湊的價值在「任兩段比較」這種 KMP 做不到的場景

## 怎麼選

| 要做的事 | 用什麼 |
| --- | --- |
| 找樣式出現位置 | KMP 或 Z，都 O(n+m) 確定性 |
| 最長 border／最小週期／重複子字串 | KMP 的 `pi` |
| 「從這裡開始和開頭相同多久」 | Z-function |
| **任兩個子字串是否相等** | 雜湊（KMP／Z 做不到） |
| 最長回文子字串 | Manacher O(n)，或雜湊 + 二分 |
| 多個樣式同時找 | Aho-Corasick（KMP 的多模式版）或 Trie |
| 字典序、後綴相關 | 後綴陣列 / 後綴自動機 |

## 常見陷阱

> [!warning] 五個坑
> - **分隔符選到輸入裡有的字元**：搜尋結果會錯。
> - **mod 2⁶⁴ 自然溢位**：見上表。
> - **雜湊減法沒補 `+ M`**：得到負數，比較永遠不相等。
> - **`prefixFunction` 的 `while` 寫成 `if`**：只退一層，退不夠就漏匹配。
> - **拿雜湊當確定性答案**：碰撞機率再小也不是 0。能用 KMP／Z 就用。

## Related Problems

- [[0028-Find-the-Index-of-the-First-Occurrence-in-a-String]] — KMP 的原題
- [[0459-Repeated-Substring-Pattern]] — `n % (n - pi[n-1]) == 0` 一行解
- [[0214-Shortest-Palindrome]] — KMP 在 `s + '#' + reverse(s)` 上的經典技巧
- [[1044-Longest-Duplicate-Substring]] — 雜湊 + 二分長度
- [[0005-Longest-Palindromic-Substring]] — Manacher，或雜湊 + 二分
- [[Modular-Arithmetic]] — 雜湊的模運算基礎
