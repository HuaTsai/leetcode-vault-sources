---
tags:
  - cpp
  - performance
  - tooling
dg-publish: true
---

## 這篇在解決什麼

[[Micro-Optimization-Myths]] 是「你對成本的直覺是錯的」，它每條結論都附實測。這篇是那些實測**怎麼跑出來的**——工具指令、降雜訊手法，以及最重要的：**怎麼判斷你量到的是真訊號還是機器在跟你開玩笑**。

順序是有意義的：先讓數字站得住腳（不需要任何工具），再上 profiler。反過來做的話，你會用很精密的工具量一個從一開始就設計壞掉的實驗。

> [!warning] 先講適用範圍
> 這篇的東西在 LeetCode 尺度（n ≤ 10⁵、單次執行幾毫秒）多半量不出差異——那個規模下判題雜訊遠大於真實差距。它的用途是**當你想弄清楚「為什麼這樣寫比較快」時，有辦法拿出證據**，而不是拿來調 LeetCode 分數。

本篇所有輸出來自：12th Gen i5-12500H、g++ 14.3.0、valgrind 3.22.0。cache 規格用 `getconf` 讀到的是 L1d 32 KB／核、L2 2 MB／核、L3 18 MB。

## 一、先讓數字站得住腳（不需要工具的部分）

這節比後面所有工具都重要。以下每條都是**做錯就會量到假東西**的。

### 防止編譯器把你要測的東西整段刪掉

回傳值沒人用，`-O2` 會直接把呼叫消掉，然後你量到 0.2 ns 並得出「這個演算法快得驚人」的結論。

```cpp
volatile bool sink;              // 全域 volatile，編譯器不准省略對它的寫入
...
sink = isSameTree(p, q);         // 迫使函式真的被執行
```

### 交錯測量，不要分組測量

```cpp
// ✗ 錯：把 A 跑 20 次、再把 B 跑 20 次
//    A 跑的時候 CPU 還涼、跑到 B 已經降頻，差距全是溫度造成的
for (int r = 0; r < 20; ++r) sink = A();
for (int r = 0; r < 20; ++r) sink = B();

// ✓ 對：同一輪裡依序各跑一次，跑 20 輪
//    頻率漂移、排程干擾被平均分攤到每個受測對象
for (int r = 0; r < 20; ++r) {
  t0 = now(); sink = A();
  t1 = now(); sink = B();
  t2 = now();
  if (r == 0) continue;          // 第一輪當 warm-up 丟掉
  ta.push_back(t1 - t0);
  tb.push_back(t2 - t1);
}
```

### 取最小值，不要取平均

雜訊**只會讓程式變慢，不會讓它變快**——干擾是單向的。所以最小值才是「這段程式在沒被打擾時的真實成本」，平均值反映的是你機器當下有多忙。中位數是折衷，比平均好，仍不如最小值穩。

### 一定要把離散度印出來

```cpp
auto spread = [](vector<double> v){
  sort(v.begin(), v.end());
  return (v.back() - v.front()) / v[v.size()/2] * 100;   // (max-min)/median
};
```

> [!important] 離散度大於差距本身時，你沒有結論，只有雜訊
> 實例：某次量到「A 比 B 慢 1.41x」，但兩者離散度分別是 ±34% 和 ±63%——這個 1.41x 完全不能用。**處理方式不是取更多樣本後硬下結論，是先去解決雜訊來源**（見下一條），或改用不受頻率影響的指標（第三節的 cachegrind）。

### 綁核心，並確認頻率調節器

```bash
taskset -c 2 ./bench            # 綁到 2 號核心，避免被搬到別的核心而丟光 cache
cpupower frequency-info         # 不需 root。看 governor 和頻率範圍
```

`cpupower` 的輸出裡這兩行是雜訊的元兇：

```txt
  hardware limits: 400 MHz - 3.30 GHz
  current policy: ... The governor "powersave" may decide which speed to use
```

`powersave` 治理器會讓頻率在 400 MHz 到 3.3 GHz 之間跳，**8 倍的浮動**——這就是離散度 ±60% 的來源。要壓下來得 root：

```bash
sudo cpupower frequency-set -g performance      # 暫時切到 performance
```

現代 Intel 混合架構（P-core／E-core）還有一層陷阱：**被排到 E-core 上跑，結果會系統性偏慢**。`taskset` 綁一顆固定的核心至少讓這件事變成常數而不是隨機。

## 二、工具速查

| 想知道 | 指令 | 要 root？ |
|---|---|---|
| 兩個**執行檔**誰快 | `hyperfine -N --warmup 3 './a' './b'` | 否 |
| 記憶體峰值／page fault／context switch | `/usr/bin/time -v ./prog` | 否 |
| cache 規格（**每核心**真實值） | `getconf -a \| grep -i cache` | 否 |
| cache miss、指令數、分支誤判 | `valgrind --tool=cachegrind --branch-sim=yes ./prog` | 否 |
| 編譯器實際生出什麼指令 | `g++ -O2 -S -masm=intel -o - x.cpp` | 否 |
| 反組譯已編好的二進位 | `objdump -d -M intel --no-show-raw-insn x.o` | 否 |
| 真實硬體計數器 | `perf stat -e cycles,instructions ./prog` | **是**（見第四節） |
| CPU 頻率治理器狀態 | `cpupower frequency-info` | 否（要改才需要） |

> [!tip] 查 cache 大小用 `getconf`，不要用 `lscpu`
> `lscpu` 印的是**全機總和**：`L1d cache: 448 KiB (12 instances)`——你得自己知道要除以 12，而且混合架構下 P-core 和 E-core 大小不同，除出來還是錯的。`getconf` 給的是**當前核心**的值，可直接用：
>
> ```bash
> $ getconf -a | grep -i cache | grep -v ' 0$'
> LEVEL1_DCACHE_SIZE                 32768
> LEVEL1_DCACHE_LINESIZE             64
> LEVEL2_CACHE_SIZE                  2097152
> LEVEL3_CACHE_SIZE                  18874368
> ```
>
> `LINESIZE 64` 也是常用的數字——它決定「一次 cache miss 會順便載進來多少相鄰資料」，是所有資料佈局優化的基本單位。

### `hyperfine`：比較整個執行檔

```bash
hyperfine -N --warmup 3 -r 20 './bench a' './bench b'
```

`-N` 是關鍵：不經過 shell，否則 shell 啟動時間（約幾 ms）會蓋掉你要測的東西。它自己會警告：

```txt
Warning: Command took less than 5 ms to complete. ... You can try to use the `-N`/`--shell=none` option
```

適用場合是**整支程式**的比較（不同編譯選項、不同輸入檔）。要比同一支程式裡的兩個函式，還是得自己寫計時迴圈——`hyperfine` 量不到函式層級。

### `/usr/bin/time -v`：最便宜的記憶體與干擾檢查

注意要寫完整路徑，`time` 這個字會被 shell 內建指令搶走。

```bash
$ /usr/bin/time -v ./bench 2>&1 | grep -Ei 'Maximum resident|page faults|context switch'
	Maximum resident set size (kbytes): 36240
	Major (requiring I/O) page faults: 0
	Minor (reclaiming a frame) page faults: 9467
	Voluntary context switches: 1
	Involuntary context switches: 1
```

用途有二：**Maximum RSS** 直接驗證你的空間複雜度分析對不對；**Involuntary context switches** 如果是個大數字，代表你的量測一直被搶走 CPU，那批數據不用看了。

## 三、cachegrind：不需要特權的硬數據（首選）

`perf` 常常被權限鎖住（第四節），而 `cachegrind` 是**純軟體模擬**，任何人都能跑：

```bash
valgrind --tool=cachegrind --cache-sim=yes --branch-sim=yes ./prog
```

> [!warning] `--cache-sim=yes` 不能省
> valgrind 3.22 的 cachegrind 預設**只印 `I refs` 一行**，cache 的部分完全不會出現。第一次跑很容易以為工具壞了。`--branch-sim=yes` 同理，要分支預測數據就得明寫。

輸出長這樣，要看的是 `D1`（L1 資料 cache）和 `LLd`（最後一層資料 cache）：

```txt
I  refs:        9,732,384          ← 指令數。純指令成本的直接度量
D  refs:        4,501,187  (2,301,836 rd + 2,199,351 wr)
D1  misses:       226,982          ← L1 資料 miss
LLd misses:        42,710          ← 掉到記憶體的次數，最貴的那個
D1  miss rate:        5.0%
Branches:       1,815,479  (1,799,879 cond + 15,600 ind)
Mispredicts:       44,089  (   40,851 cond +  3,238 ind)
Mispred rate:         2.4%
```

### 判讀方式：拿它來**分離**原因，而不是預測時間

這是 cachegrind 真正的價值。時間只給你一個混合了所有因素的數字，cachegrind 讓你知道**差距來自哪裡**。判讀規則：

- **`I refs` 差很多、`D1/LLd` 幾乎相同** → 差距是**指令成本**（資料結構的存取邏輯更繁瑣、額外的間接定址、邊界檢查）。這種差距**不隨資料量變化**。
- **`I refs` 幾乎相同、`D1/LLd` 差很多** → 差距是**記憶體存取模式**（佈局、走訪順序、工作集大小）。這種差距**隨資料量放大**。
- **`Mispredicts` 差很多** → 分支預測問題，詳見 [[Micro-Optimization-Myths]]。

一組真實的分離範例（同一棵樹、同樣的走訪順序，只換底層容器）：

```txt
方法                    I refs      D1 miss     LLd miss
stack<deque>        29,858,950      595,984      124,171
stack<vector>       20,291,689      596,000      124,172
```

cache miss **一模一樣**（差 16 次），指令數差 47%——結論非常乾淨：`deque` 的成本完全是指令（查 map array → 定位 chunk → 邊界檢查），和 cache 一點關係都沒有。光看時間永遠得不到這個結論。

> [!warning] cachegrind 的三個限制
> 1. **慢 20–50 倍**。測資要縮小，但別縮到工作集塞進 L1，否則你要看的效應就消失了。
> 2. **模擬的是理想 cache**：不模擬硬體預取器、不模擬亂序執行與記憶體平行度。所以它算出的 miss 次數**不能拿來換算時間**——順序存取的 miss 在真機上會被預取器藏掉大半，隨機存取則不會。
> 3. 預設只模擬 L1 和 LL 兩層，中間的 L2 被忽略。

## 四、perf：能用的話最準，但常常被鎖住

`perf` 讀的是真實硬體計數器，沒有模擬誤差。問題是預設權限：

```bash
$ perf stat -e cycles,instructions ./prog
Error:
No supported events found.
Access to performance monitoring and observability operations is limited.
...
perf_event_paranoid setting is 4:
```

```bash
cat /proc/sys/kernel/perf_event_paranoid     # 看目前值
```

| 值 | 允許範圍 |
|---|---|
| `-1` | 幾乎全部 |
| `0` | 禁 raw／ftrace tracepoint |
| `1` | 禁 CPU 全域事件（自己的程序仍可量） |
| `2` | 禁 kernel profiling |
| `3`／`4` | 逐步收緊，Ubuntu 近版預設 `4` |

> [!important] `paranoid=4` 連軟體事件都擋
> 值得單獨記一筆：一般會以為 `task-clock`、`page-faults`、`context-switches` 這些**軟體**事件不碰 PMU 所以應該能用——實測 `paranoid=4` 下它們一樣被拒。也就是說 `perf` 在這個設定下是**完全不可用**的，沒有「至少能量一點」的中間地帶。這時候的替代品是第三節的 cachegrind（指令與 miss）加 `/usr/bin/time -v`（page fault 與 context switch）。

要放寬（需要 root，且是降低系統安全邊界，自己斟酌）：

```bash
sudo sysctl -w kernel.perf_event_paranoid=1                        # 暫時，重開機失效
echo 'kernel.perf_event_paranoid=1' | sudo tee /etc/sysctl.d/99-perf.conf   # 永久
```

開得起來之後的常用事件：

```bash
perf stat -e cycles,instructions,cache-references,cache-misses,\
LLC-load-misses,L1-dcache-load-misses,branches,branch-misses ./prog

perf stat -x, -e cycles,instructions ./prog     # CSV 輸出，方便 script 解析
perf record -g ./prog && perf report            # 找熱點函式
```

`instructions / cycles` 就是 IPC。IPC 低（< 1）通常代表在等記憶體或分支一直猜錯；IPC 高（> 3）代表計算密集、管線跑滿。

## 五、沒有任何 profiler 時：size-scaling 法

這招不需要任何工具，只需要一個計時迴圈，而且**能回答「這個差距是不是 cache 造成的」**——常常比 profiler 更直接。

做法：**固定所有變因，只掃描資料量**，讓它跨過 L1 → L2 → L3 → RAM 的邊界，看比值怎麼變。

```txt
n         資料量      A ns/op   B ns/op    B/A
127        3 KB  (L1)    2.45      2.40   0.98x   ← 全在 L1，兩者無差別
8191     192 KB  (L2)    2.75      3.64   1.32x
131071     3 MB  (L3)    2.81      3.83   1.36x
2097151   48 MB  (RAM)   3.04      4.86   1.60x   ← 差距隨工作集放大
```

判讀規則只有兩條：

- **比值不隨 n 變化** → 是**指令成本**。恆定的比例是它的簽名。
- **比值隨 n 單調放大** → 是 **cache**。從 1.0x 開始隨工作集爬升是它的簽名。

> [!tip] 這招順便會告訴你「在你關心的規模上，這件事重不重要」
> 上表最有價值的其實是第一行：n = 127 時比值 0.98x，**代表在小資料上這個優化根本不存在**。如果你的實際輸入就是那個規模，量到大資料上的 1.60x 對你毫無意義。這比任何 profiler 的輸出都更能阻止你做無用的優化。

## 六、控制變因的手法

實驗設計錯了，工具再準也沒用。三個常用手法：

### 用 arena 精確控制記憶體佈局

想比較「走訪順序」的影響，就必須讓**節點的記憶體位置**變成可控變因，否則你量到的是 `new` 當天的心情：

```cpp
vector<TreeNode> arena(n);            // 一次配置，所有節點都住在這裡
vector<long> slot(n);                 // 邏輯編號 → arena 位置
// slot 填 level-order／preorder／shuffle，就得到三種佈局
// 三棵樹「邏輯上完全相同」，只有記憶體位置不同 → 佈局成了唯一變因
```

### 容器提到迴圈外重用，消除配置成本

否則你量的是 `malloc`，不是演算法：

```cpp
deque<P> buf;                          // 提到函式外
bool solve(...) { buf.clear(); ... }   // clear() 保留已配置的容量
```

`vector` 還要記得 `reserve`。

> [!warning] 忘記 reserve 會製造「非單調」的假訊號
> 實際踩過：某組數據在 n = 8191 量到 13.99 ns，n = 131071 反而只有 6.81 ns——**資料變大 16 倍卻變快一倍**。原因是小 n 時外層重複次數多，每次都從零開始讓 `vector` 反覆 realloc + 搬移，量到的是配置成本。**看到非單調就先懷疑實驗設計，不要急著解釋現象。**

### 一次只改一個變因

想同時知道「容器」和「走訪順序」各自的貢獻，就做 2×2，不要只比對角線。最漂亮的對照是**同一個容器只換一個操作**：

```cpp
deque<P> dq;
// DFS：dq.back()  + dq.pop_back()
// BFS：dq.front() + dq.pop_front()
```

同一個型別、同樣的 push、只差 pop 哪一端——這樣量到的差距**只可能**來自走訪順序，容器實作被完美消掉。

## 七、正確性也要工具（量之前先確定它是對的）

```bash
g++ -std=c++20 -fsanitize=address,undefined -g -o prog x.cpp   # 越界／UB
g++ -std=c++20 -D_GLIBCXX_DEBUG -g -o prog x.cpp               # STL 迭代器與索引檢查
```

`-D_GLIBCXX_DEBUG` 常被忽略，但它抓 `v[i]` 越界、失效的迭代器比 asan 更早也更清楚（直接指出是哪個 STL 操作違規）。兩者都會拖慢執行，**只在驗證階段開，量效能時務必關掉**。

比 sanitizer 更有效的是**隨機交叉驗證**：寫兩個以上的獨立實作，餵幾十萬組隨機輸入比對結果。它抓得到 sanitizer 完全看不見的邏輯錯誤（不越界、不 UB，就只是答案錯）。

### 直接編譯筆記裡的 code block

避免「筆記寫一套、驗證跑另一套」——直接把 `.md` 裡的 code 抽出來編：

````bash
awk '/^```cpp$/{f=1;next} /^```$/{f=0} f' note.md > extracted.cpp
csplit -s -z -f part -b '%d.inc' extracted.cpp '/^class Solution/' '{*}'
````

再把每個 `part*.inc` 各自 `#include` 進不同 namespace，就能在同一支測試程式裡交叉比對同一篇筆記的所有解法。**驗證的是筆記本身，不是它的副本。**

## Related Problems

[[Micro-Optimization-Myths]] — 姊妹篇。那篇是量測的**結論**（什麼貴什麼不貴），這篇是量測的**方法**
[[STL-Pitfalls]] — 第七節那些正確性工具最常抓到的東西都收在那裡
[[Tree-Traversal-Iterative]] — 第六節「同一個 deque 只換 pop 端」的對照法，出處就是那組走訪骨架
