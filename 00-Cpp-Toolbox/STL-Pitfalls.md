---
tags:
  - cpp
  - stl
  - pattern
dg-publish: true
---

## 這篇在解決什麼

C++ 的 STL 幾乎不用型別系統擋你——**編得過完全不代表對**。這篇收錄的不是「會被編譯器抓到的錯」，而是：編譯乾淨、看起來合理、跑起來也不一定崩，但**複雜度或語意是錯的**。

四類，每條都附實測數字。判斷法則在最後一節。

## 一、泛型演算法 vs 容器成員：同名但完全不同

`<algorithm>` 裡的泛型演算法**只看得到迭代器**，看不到背後是雜湊表還是紅黑樹。所以當容器提供了同名的成員函式時，兩者做的事差很多。

### `lower_bound` / `upper_bound`

|                                           | 做的事               | 複雜度           |
| ----------------------------------------- | -------------------- | ---------------- |
| `m.lower_bound(x)`（成員）                | 沿紅黑樹往下走       | O(log n)         |
| `std::lower_bound(m.begin(), m.end(), x)` | 對迭代器區間泛型二分 | **看迭代器類別** |

實測（每次查詢平均耗時）：

```txt
       n | std::lower_bound 於 map      | std::lower_bound 於 vector
         | 比較次數    每次查詢         | 比較次數    每次查詢
   25000 |     14.7      141.4 us       |     14.7      0.058 us
   50000 |     15.7      261.2 us       |     15.7      0.065 us
  100000 |     16.7      533.6 us       |     16.7      0.066 us
  200000 |     17.7     1058.8 us       |     17.7      0.077 us

log2(n)：25k=14.6  50k=15.6  100k=16.6  200k=17.6
```

**比較次數兩邊一模一樣、都精準等於 log₂(n)**，但 map 那欄的時間隨 n 加倍再加倍，vector 那欄幾乎平坦。所以慢的不是比較，是**移動迭代器**。

看 libstdc++ 的實作就懂了：

```cpp
_DistanceType __len = std::distance(__first, __last);   // ← 對 map 是 O(n)
while (__len > 0) {
  _DistanceType __half = __len >> 1;
  _ForwardIterator __middle = __first;
  std::advance(__middle, __half);                       // ← 對 map 是 O(half)
  if (__comp(__middle, __val)) { __first = __middle; ++__first; __len -= __half + 1; }
  else __len = __half;
}
```

兩個 O(1) 假設在 map 上都不成立：

- `std::distance` 對 bidirectional 迭代器得整段走過才知道長度 → **O(n)**
- `std::advance(it, k)` 只能 `++` k 次 → **O(k)**

移動總量是 `n` + `n/2 + n/4 + n/8 + …` = **O(n)**，而比較只有 log n 次。

> [!warning] 標準只保證「比較次數」，沒保證迭代器操作次數
> 文件上寫的是「至多 `log₂(last − first) + O(1)` 次**比較**」——看到 log₂ 就以為是 O(log n)，但那句話只在 random access 迭代器上等價於總工作量。**這就是這個陷阱最會騙人的地方。**

### `find`

比 `lower_bound` 更兇，因為它連「比較次數 log n」這個安慰獎都沒有，就是純線性掃：

```txt
set / unordered_set 20 萬筆，每次查詢平均：
  s.find(x)                            0.290 us     成員版 O(log n)
  std::find(s.begin(), s.end(), x)   469.020 us     泛型版 O(n)     慢 1600 倍
  us.find(x)                           0.017 us     成員版 O(1)
  std::find(us.begin(), us.end(), x) 250.871 us     泛型版 O(n)     慢 15000 倍
```

### 對照清單

| 泛型版（O(n) 或退化） | 成員版                 | 成員版複雜度              |
| --------------------- | ---------------------- | ------------------------- |
| `std::find`           | `.find()`              | O(log n) ／ O(1) 均攤     |
| `std::count`          | `.count()`             | O(log n + k) ／ O(1) 均攤 |
| `std::lower_bound`    | `.lower_bound()`       | O(log n)                  |
| `std::upper_bound`    | `.upper_bound()`       | O(log n)                  |
| `std::equal_range`    | `.equal_range()`       | O(log n)                  |
| `std::binary_search`  | `.contains()`（C++20） | O(log n) ／ O(1) 均攤     |

適用容器：`map` / `multimap` / `set` / `multiset` / `unordered_*`。

> [!important] 現代寫法反而更容易踩
> 舊寫法要手動掏 `begin()/end()`，多少會停頓一下。但 ranges 版可以直接吃容器，看起來又短又現代：
>
> ```cpp
> ranges::lower_bound(m, 5, {}, &pair<const int, string>::first);   // 編得過，O(n)
> m.lower_bound(5);                                                  // O(log n)
> ```
>
> **判準不是「用哪套 API」，而是「容器有沒有自帶同名成員」。**有就一律用成員版。

## 二、演算法的前提沒滿足：編得過，但是 UB

### comparator 必須是嚴格弱序

最核心的要求是 **`comp(a, a)` 必須為 false**。寫成 `<=` 是最常見的違反：

```cpp
vector<int> v(50);
for (auto& x : v) x = rng() % 3;                               // 大量重複值
sort(v.begin(), v.end(), [](int a, int b) { return a <= b; }); // 注意 <=
```

```txt
正常編譯執行 → Segmentation fault (core dumped)，退出碼 139
```

**不是「順序怪怪的」，是直接記憶體越界。**libstdc++ 的 introsort 內層迴圈靠 `comp` 回傳 false 當哨兵停下來，`<=` 讓相等元素永遠回傳 true，指標就衝出陣列。重複值越多越容易觸發（上例只有三種值）。

排序想要「相等時依另一個欄位」，正確做法是用 `tuple` 的字典序或明確處理平手，而不是把 `<` 放寬成 `<=`：

```cpp
sort(v.begin(), v.end(), [](const auto& a, const auto& b) {
  return tie(a.score, a.name) < tie(b.score, b.name);   // 仍是嚴格弱序
});
```

### 解方：用 `<=>` 讓編譯器幫你寫比較運算子

上面那個坑的根源是「你在手寫比較邏輯」。C++20 的三路比較可以整個省掉：

```cpp
struct Node {
  int a; string b;
  auto operator<=>(const Node&) const = default;   // 逐成員字典序
};
```

一行取代六個運算子，而且**編譯器生成的一定是合法的嚴格弱序**——你不可能不小心把 `<` 寫成 `<=`，因為你根本沒在寫比較邏輯。定義完直接能丟進 `sort` / `set` / `priority_queue`，不用任何 comparator。

**效能沒有代價。**200 萬筆三欄位 struct 排序實測：

```txt
  手寫 operator<        161.4 ms
  預設 operator<=>      165.2 ms      差 2%，在測量誤差內
```

編譯器對 defaulted 版本一樣會生成短路比較，不是真的算完整個三路結果再比。

> [!warning] `==` 只有 `= default` 才會附贈
>
> ```txt
> 預設 <=>：  <  1   >=  1   ==  1
> 自訂 <=>：  <  1   >=  1   ==  0   ← == 沒有被生出來
> ```
>
> 規則是：**defaulted `<=>` 會隱式宣告 `operator==`；自訂的 `<=>` 不會。**標準這樣設計是因為 `==` 往往能寫得比 `<=>` 快（例如先比長度就能否決），所以不從三路比較推導。
>
> ```cpp
> struct Custom {
>   int a; string b;
>   auto operator<=>(const Custom& o) const { return a <=> o.a; }   // 只比 a
>   bool operator==(const Custom& o) const { return a == o.a; }     // ← 必須自己補
> };
> ```
>
> 忘了補的話，要等到用 `find` / `count` / `unordered_set` 時才會炸出來。

> [!note] 有 `double` 成員會變成 `partial_ordering`
>
> ```txt
> int + string  -> strong_ordering
> int + double  -> partial_ordering
> ```
>
> `partial_ordering` 表示「可能無法比較」，來源是 NaN：
>
> ```txt
> NaN < 1.0 = 0,  NaN > 1.0 = 0,  NaN == 1.0 = 0   ← 三者皆假
> ```
>
> 這違反嚴格弱序要求的「不可比較性必須可遞移」，所以**含 NaN 的資料丟給 `sort` 是 UB**。這不是 `<=>` 引進的問題（手寫 `operator<` 一樣中招），但 `<=>` 的好處是**型別會誠實告訴你**——看到回傳 `partial_ordering` 就該警覺。

> [!important] 不要把「這次想要的順序」烙進型別
> `operator<` 應該表達**這個型別本身的自然序**，一次性的排序規則留在呼叫點：
>
> ```cpp
> // ✗ 把「依分數降序」寫死在型別裡，換個地方想升序就完蛋
> struct P { int score; auto operator<=>(const P& o) const { return o.score <=> score; } };
>
> // ✓ 排序規則留在呼叫點，同一份資料換 key 互不干擾
> ranges::sort(v, greater{}, &P::score);
> ranges::sort(v, {}, &P::name);
> ```
>
> 「依某欄位排」用 projection，不要為此改型別的比較運算子。只需要相等比較時也一樣——只寫 `bool operator==(const T&) const = default;`，別引入 `<=>`，多一個順序就多一個會被誤用的介面。

### 二分要求區間已排序

`lower_bound` / `upper_bound` / `binary_search` 都要求區間**已依同一個序排好**。沒排序不會報錯、不會崩，只會安靜給錯答案——而且錯得不穩定，隨資料而異。

> [!tip] 本機練習加 `-D_GLIBCXX_DEBUG`
> 上面那個 `<=` 在正常編譯下是段錯誤（看不出錯在哪），加了 `-D_GLIBCXX_DEBUG` 會直接 abort 並指出是 `std::sort` 的 comparator 有問題。完整的除錯旗標組合見本篇最後的〈工具〉一節。

## 三、演算法不做你以為的事

`remove` 和 `unique` **都不會刪除任何元素**。容器的 `size` 不歸演算法管——它們只看得到迭代器，改不了容器大小。它們做的是「把要保留的往前搬，回傳新的邏輯結尾」：

```txt
原始                      size=6  [1,2,3,2,4,2]
remove(2) 之後            size=6  [1,3,4,2,4,2]   ← size 沒變，尾巴是垃圾
再 erase(it, end())       size=3  [1,3,4]

unique 之後               size=6  [1,2,3,2,2,3]   ← 同樣沒變
再 erase                  size=3  [1,2,3]
```

漏掉第二步就會留下一段垃圾，而且因為 `size` 沒變，後面的 loop 照樣會讀到它。

> [!tip] C++20 起直接用 `std::erase` / `std::erase_if`
>
> ```cpp
> v.erase(remove(v.begin(), v.end(), 2), v.end());   // 舊：erase-remove idiom
> erase(v, 2);                                        // 新：一行，沒有中間狀態可以漏
> erase_if(v, [](int x) { return x % 2 == 0; });
> ```
>
> 對 `map` / `set` 也適用，而且是**唯一**正確的批次刪除方式（`std::remove` 根本沒辦法用在關聯容器上——`value_type` 的 key 是 const，搬不動）。

另外 `unique` 只去除**相鄰**的重複，沒先排序等於沒作用。

## 四、有副作用的操作

### `map::operator[]` 找不到會插入

```txt
查詢前 size=1
if (m["nope"] == 0) ...           ← 只是想「查查看」
查詢後 size=2                      ← 憑空多了 {"nope", 0}
m.contains("nope") = 1             ← 現在它真的存在了
```

**查詢一律用 `contains` / `find` / `at`，`operator[]` 只留給「我就是要寫入」。**這也是為什麼用了 `operator[]` 的函式沒辦法宣告成 `const`——它是非 const 多載。`at` 有 const 多載，語意也更誠實：「我確定它在，不在就是邏輯錯誤」。

### 邊走邊 erase

`erase` 之後迭代器已失效，再 `++it` 是 UB：

```cpp
for (auto it = m.begin(); it != m.end(); ) {
  if (pred(*it)) it = m.erase(it);   // erase 回傳下一個，接住它
  else ++it;                          // 注意 for 的第三格是空的
}
erase_if(m, pred);                    // C++20，一行取代上面整段
```

失效範圍各容器不同，`vector` 最兇：

| 容器          | `erase` 後失效範圍                     |
| ------------- | -------------------------------------- |
| `map` / `set` | 只有被刪的那一個                       |
| `list`        | 只有被刪的那一個                       |
| `vector`      | **刪除點之後全部**                     |
| `unordered_*` | 只有被刪的那一個（但 rehash 會全失效） |

`vector` 另外要注意 `push_back` 觸發擴容時，**所有**迭代器、指標、參考全部失效。

## 五、型別本身在騙你

前四類是「用錯了 API」，這一類是「型別長得像 A，其實是 B」。

### `vector<bool>` 不是裝 bool 的 vector

它是位元打包的**特化**，`operator[]` 回傳的是一個 proxy 物件，不是 `bool&`：

```txt
decltype(vb[0]) 是 bool& 嗎？ 0
decltype(vc[0]) 是 char& 嗎？ 1   <- vector<char> 對照組
```

最陰險的後果是 `auto` 會抓到 proxy，而 proxy 對容器有寫入能力：

```cpp
vector<bool> vb{false, false, false};
auto x = vb[0];        // x 不是 bool，是 _Bit_reference
x = true;              // 看起來只改了區域變數⋯⋯
```

```txt
x = true 之後 vb[0] = 1        <- 竟然改到了容器裡
同樣寫法用 vector<char>：vc[0] = 0   <- 不受影響
```

其他症狀：`bool& r = v[0];` 編不過（`cannot bind non-const lvalue reference to an rvalue`）、拿不到 `bool*`、不滿足標準的 Container 要求。

**效能上也不划算**，因為每次存取都要位移加遮罩。拿 [[0003-Longest-Substring-Without-Repeating-Characters]] 的 256 格查表實測 300 萬字元的輸入：

```txt
  vector<bool>  15.5 ms
  array<bool>   12.4 ms      快 20%
```

省下來的只有記憶體：256 格用 `vector<bool>` 是 32 bytes、`array<bool, 256>` 是 256 bytes——在 L1 面前這個差距毫無意義。

> [!tip] 替代品
>
> - 固定大小的旗標表 → `array<bool, N>` 或 `bitset<N>`（`bitset` 同樣位元打包，但介面誠實，而且有 `count()` / `any()` / 位元運算）
> - 需要真正的動態 `bool` 容器 → `vector<char>` 或 `vector<uint8_t>`
> - **真的想省記憶體且量夠大時**才用 `vector<bool>`，並且避開 `auto`

### `char` 的符號性是實作定義的

`char`、`signed char`、`unsigned char` 在 C++ 裡是**三個不同的型別**——不像 `int` 和 `signed int` 是同一個：

```txt
char 與 signed char 同型？   0
char 與 unsigned char 同型？ 0
int  與 signed int   同型？  1   <- 對照組
```

標準沒規定 `char` 的符號性，各平台 ABI 自己決定：

| 平台                                 | `char`   |
| ------------------------------------ | -------- |
| x86-64 Linux / macOS / Windows       | **有號** |
| ARM32 / AArch64 **Linux**            | **無號** |
| **Apple Silicon**（macOS/iOS ARM64） | **有號** |
| Windows on ARM（MSVC）               | **有號** |

「ARM 就是無號」在 M 系列 Mac 上會踩空——Apple 刻意保持有號以相容他們的 x86 程式碼。而且這是 **ABI 的決定，不是 CPU 的性質**，同一台機器用旗標就能翻轉：

```txt
預設              char 是有號，範圍 -128 ~ 127   (0xE4 讀成 -28)
-funsigned-char   char 是無號，範圍 0 ~ 255      (0xE4 讀成 228)
```

> [!warning] 拿字元當陣列索引一律轉 `(unsigned char)`
> 有號的 `char` 遇到位元組值 `≥ 128` 會變成負數（`-128 ~ -1` 對應 `128 ~ 255`），拿去當索引就是**負索引越界**：
>
> ```txt
> arr[(unsigned char)c]  →  存取 arr + 228     合法
> arr[c]                 →  存取 arr - 28      越界
> ```
>
> 不是「讀到錯的格子」，是讀到陣列以外的記憶體——ASan 報的是 `heap-buffer-overflow`。
>
> 轉型是**零成本**的：位元完全沒動，只是改變解讀方式（`11100100` 讀成 `-28` 或 `228`）。所以別去推理「這台機器的 `char` 是不是有號」，一律轉。

> [!note] 為什麼會有這個歷史差異
> 關鍵在**哪種載入位元組的指令比較便宜**。
>
> 早期 ARM（ARMv1～v3）**沒有有號位元組載入指令**，只有零延伸的 `LDRB`；要符號延伸得寫三條指令（載入、左移 24、算術右移 24）。`LDRSB` 到 ARMv4 才出現。所以 `char` 設成無號讓每次載入從三條變一條，差距大到不可能選另一邊；ABI 定了之後即使有了 `LDRSB` 也不能改。
>
> x86 兩種都有（`MOVSX` / `MOVZX`），單指令、都便宜，硬體沒給偏好，於是純粹繼承歷史——K&R C 在 PDP-11 上開發，該機器 `MOVB` 載入時符號延伸，`char` 自然是有號。
>
> 一句話：**ARM 是被指令集逼的，x86 是繼承 PDP-11 的。**

## 工具：本機除錯的編譯旗標

這些陷阱編譯器預設不會擋，但**開了旗標就會**。

```bash
g++ -std=c++20 -g -O1 -fsanitize=address,undefined -fno-omit-frame-pointer \
    -D_GLIBCXX_DEBUG -Wall -Wextra main.cpp
```

| 旗標                      | 作用                                                       |
| ------------------------- | ---------------------------------------------------------- |
| `-fsanitize=address`      | 越界、use-after-free、double-free、記憶體洩漏              |
| `-fsanitize=undefined`    | 有號溢位、位移越界、空指標解參考、對齊錯誤                 |
| `-D_GLIBCXX_DEBUG`        | 容器語意：索引超過 `size()`、迭代器失效、comparator 不合法 |
| `-g`                      | 報告裡才會有**行號**                                       |
| `-fno-omit-frame-pointer` | 堆疊追蹤才讀得懂                                           |

實際輸出（把 `char` 直接當索引的那個 bug）：

```txt
ERROR: AddressSanitizer: heap-buffer-overflow
    #0 in std::_Bit_reference::operator bool() const stl_bvector.h:87
    #1 in lengthOfLongestSubstring(...) demo.cpp:7      ← 直接指到那一行
    #2 in main demo.cpp:15
```

> [!important] Sanitizer 和 `_GLIBCXX_DEBUG` 抓的東西互補，要一起開
>
> - **ASan 看記憶體**——超出配置範圍才會報。`vector` 的 `operator[]` 越界如果還在 capacity 內，那塊記憶體是合法的，ASan 看不到。
> - **`_GLIBCXX_DEBUG` 看容器語意**——索引超過 `size()`、迭代器失效、comparator 違反嚴格弱序，這些 ASan 一律無感。
>
> 少開任何一個都會有盲區。

常用的執行期環境變數：

```bash
UBSAN_OPTIONS=print_stacktrace=1    # UBSan 預設只印一行，這個才會給堆疊
ASAN_OPTIONS=abort_on_error=1       # 出錯立刻 abort，方便配 gdb
```

> [!warning] 只在本機除錯用
> ASan 約慢 2 倍、記憶體約 3 倍，`_GLIBCXX_DEBUG` 再慢幾倍。**別拿這組旗標測效能**，也別交到 judge 上。另外 `-fsanitize=address` 和 `-fsanitize=thread` 互斥，不能同時開。

fish 使用者可以包成 function 省得每次打：

```fish
function gpp
    g++ -std=c++20 -g -O1 -fsanitize=address,undefined -fno-omit-frame-pointer \
        -D_GLIBCXX_DEBUG -Wall -Wextra $argv
end
```

## 自我檢查的四個問句

這些陷阱的共同點是編譯器**預設**不會幫你擋。動 STL 前問自己：

1. **這個容器有沒有自帶同名成員？** 有就用成員版。（第一類）
2. **這個演算法的前提我滿足了嗎？** 已排序？comparator 是嚴格弱序？——自訂型別的話，`= default` 的 `<=>` 讓這題免答。（第二類）
3. **它會改變 size 嗎？會有副作用嗎？** `remove`/`unique` 不會改 size；`operator[]` 會插入。（第三、四類）
4. **這個型別真的是它看起來的樣子嗎？** `vector<bool>` 不是 bool 容器；`char` 的符號性因平台而異。（第五類）

問不出來的時候，就把旗標開起來讓機器回答。

## Related Problems

[[0981-Time-Based-Key-Value-Store]] — `lower_bound` / `upper_bound` 的 comparator 參數順序、`std::` 與 `std::ranges::` 的差異、projection 用法都在該篇的附錄
[[Binary-Search-Templates]] — 決定要不要自己手寫二分之前先看這篇；能用 STL 就別手寫
[[0153-Find-Minimum-in-Rotated-Sorted-Array]] — 手寫二分的邊界推導範例，對照「外包給 STL」的成本效益
[[0003-Longest-Substring-Without-Repeating-Characters]] — 第五類兩條陷阱的實戰現場：`vector<bool>` 當查表、`char` 直接當索引
