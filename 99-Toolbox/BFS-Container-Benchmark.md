---
tags:
  - cpp
  - performance
  - tooling
  - graph
  - breadth-first-search
dg-publish: true
---

## 這篇在解決什麼

一個具體問題的完整實驗紀錄：**BFS 的待處理集合，該用 `queue`（底層 `deque`）還是兩個 `vector` 交替（`cur` / `next`）？**

[[Benchmarking-Toolkit]] 是方法的目錄，這篇是把那些方法從頭到尾套一次的案例——包含**兩個那篇沒收的手法**（用 callgrind 圈出程式的一小段、用 hook `operator new` 量記憶體高水位），以及一個更重要的示範：**同一組數據，合規測與不合規測會給出不同的結論。**

起因是 [[0102-Binary-Tree-Level-Order-Traversal]]。那題的樹上 `cur` / `next` 比佇列少 20% 指令，於是問題自然浮出來：這個優勢是**這題特有**，還是**所有 BFS 通用**？

環境：12th Gen i5-12500H（P/E 混合架構）、L1d 32 KB／核、L2 2 MB／核、L3 18 MB、g++ 14.3.0、valgrind 3.22.0、governor `powersave`（無 root 改不動，這點下面會反覆咬人）。

## 零、受測的兩個實作

圖用 CSR（`head` / `adj` 兩個陣列）以免鄰接表的 `vector<vector<int>>` 把記憶體佈局的變因混進來。

```cpp
long long bfs_queue(const Graph& g, int src, vector<int>& dist) {
  fill(dist.begin(), dist.end(), -1);
  queue<int> q;
  q.push(src);
  dist[src] = 0;
  long long s = 0;
  while (!q.empty()) {
    int u = q.front();
    q.pop();
    s += u;
    for (int i = g.head[u]; i < g.head[u + 1]; ++i) {
      int v = g.adj[i];
      if (dist[v] < 0) {
        dist[v] = dist[u] + 1;   // ← 要回頭讀 dist[u]
        q.push(v);
      }
    }
  }
  return s;
}

long long bfs_vec(const Graph& g, int src, vector<int>& dist) {
  fill(dist.begin(), dist.end(), -1);
  vector<int> cur{src}, next;
  dist[src] = 0;
  int d = 0;
  long long s = 0;
  while (!cur.empty()) {
    next.clear();
    ++d;                          // ← 層號免費，不必讀 dist[u]
    for (int u : cur) {
      s += u;
      for (int i = g.head[u]; i < g.head[u + 1]; ++i) {
        int v = g.adj[i];
        if (dist[v] < 0) {
          dist[v] = d;
          next.push_back(v);
        }
      }
    }
    cur.swap(next);
  }
  return s;
}
```

兩者都累加 `s` 並在測完比對，確保走訪的節點集合一致——**先證明兩邊在算同一件事，再談誰快**。

> [!note] `cur` / `next` 版順手省掉一次隨機讀取
> 佇列版算距離要 `dist[u] + 1`，那是一次對大陣列的隨機存取；`cur` / `next` 版的層號 `d` 是個暫存器裡的整數。這個差異在下面的指令數上看得到。

## 一、實驗設計：掃「圖的形狀」，不是掃「資料量」

[[Benchmarking-Toolkit]] 第五節的 size-scaling 是固定形狀、掃資料量，用來分辨「指令成本 vs cache 成本」。這題要問的不一樣：兩個實作的差別在**每層要付一筆固定開銷**（`clear()` + `swap()` + 迴圈起手式），所以真正的變因是**層數與層寬的比例**。掃資料量不會碰到這個變因，掃形狀才會。

四種圖是刻意挑的極端，每一種逼出一個不同的答案：

| 圖形 | n | 層數 | 每層寬度 | 想逼出什麼 |
|---|---|---|---|---|
| 隨機稀疏圖 | 200k，m=1M | 少 | 很寬且不規則 | 最接近真實工作負載 |
| 網格 1000×1000 | 1M | ~2000 | 數百到 2000 | 規則、局部性好的中間案例 |
| **鏈狀** | 1M | **1M** | **1** | 每層固定開銷的放大鏡 |
| **星形** | 1M | **2** | **1M** | 單層超寬，容器擴容行為的放大鏡 |

> [!important] 極端形狀不是為了「模擬真實」，是為了讓某個成本項單獨現形
> 鏈狀圖在真實世界幾乎不存在，但它把「每層固定開銷 × 層數」這一項乘上 100 萬倍，讓原本淹沒在雜訊裡的東西變成可測量的訊號。**設計實驗時挑的是能隔離變因的輸入，不是有代表性的輸入**；代表性交給隨機圖那一列。

## 二、時間：合規測與不合規測給出不同結論

先照 [[Benchmarking-Toolkit]] 第一節的四條規範寫計時迴圈——交錯測量、丟掉第一輪、取最小值、印離散度：

```cpp
volatile long long sink;                       // 防止 -O2 把整段消掉

for (int r = 0; r < 20; ++r) {
  auto t0 = steady_clock::now();
  sink = bfs_queue(g, 0, dist);
  auto t1 = steady_clock::now();
  sink = bfs_vec(g, 0, dist);                  // 同一輪內依序各跑一次
  auto t2 = steady_clock::now();
  if (r == 0) continue;                        // 第一輪當 warm-up
  tq.push_back(duration<double, milli>(t1 - t0).count());
  tv.push_back(duration<double, milli>(t2 - t1).count());
}
// 取 min，並印 (max - min) / median 當離散度
```

執行時綁核：

```bash
taskset -c 2 ./bench
```

結果（`ms` 欄是 19 次的最小值）：

| 圖形 | `queue` | 離散度 | `cur/next` | 離散度 | 比值 |
|---|---|---|---|---|---|
| random n=200k m=1M | 8.44 | ±28% | 8.36 | ±25% | 1.01x |
| grid 1000×1000 | 8.08 | ±11% | 7.61 | ±7% | 1.06x |
| **chain n=1M** | **3.85** | **±6%** | **5.26** | **±6%** | **0.73x** |
| star n=1M | 3.21 | ±35% | 3.12 | ±16% | 1.03x |

> [!warning] 這張表只有一列可以下結論
> 規則是「離散度大於差距本身時，你沒有結論」。random 的差距 1%、離散度 ±28%；star 的差距 3%、離散度 ±35%——**這兩列什麼都沒說**。grid 的 6% vs ±11% 也不夠格。只有 chain 那列差距 37%、離散度 ±6%，訊號遠大於雜訊。
>
> 離散度來自 `powersave` 治理器讓頻率在 400 MHz–3.3 GHz 之間跳，沒有 root 壓不下去；`taskset` 只能消掉「被搬到 E-core」那一部分。**這正是為什麼需要第三節。**

> [!important] 不合規的測法會生出假結論——這是實測踩到的
> 第一版我用「A 連跑 5 次、再 B 連跑 5 次」、不綁核、不印離散度，量到 star 是 **1.10x**、grid 是 1.04x，還一度寫成「寬圖上 cur/next 小勝」。合規重跑後 star 變 1.03x，而 ±35% 的離散度直接判定那個 1.10x 沒有意義。差別只在測法，數據本身沒變。**先把離散度印出來，再決定要不要相信自己的結論。**

## 三、callgrind：圈出程式的一小段來數

時間量不出來的三列，換成不受頻率影響的指標就能量。但直接跑 `cachegrind` 會連**建圖**一起數進去——建一張 100 萬節點的 CSR 比 BFS 本身貴得多，訊號會被稀釋掉。

`cachegrind` 沒有「只數這一段」的功能，`callgrind` 有：

```cpp
#include <valgrind/callgrind.h>
// ...
CALLGRIND_TOGGLE_COLLECT;          // 開始數
for (int i = 0; i < 3; ++i) s += mode ? bfs_vec(g, 0, dist) : bfs_queue(g, 0, dist);
CALLGRIND_TOGGLE_COLLECT;          // 停止數
```

```bash
valgrind --tool=callgrind --collect-atstart=no --cache-sim=yes \
         --callgrind-out-file=/dev/null ./iso <mode> <shape>
```

`--collect-atstart=no` 讓它啟動時不收集，之後由程式裡的 `CALLGRIND_TOGGLE_COLLECT` 開關。`--callgrind-out-file=/dev/null` 是因為這裡只要 stderr 上的總計，不需要那份給 `kcachegrind` 看的巨大 profile。編譯時不必連結任何東西，`valgrind/callgrind.h` 是純巨集（沒跑在 valgrind 下時會展開成 no-op）。

### 名詞：`I refs` 是什麼

| 欄位 | 意思 | 怎麼用 |
|---|---|---|
| **`I refs`** | **Instruction references**——CPU 讀取（執行）了幾條指令 | 「純指令成本」的直接度量。**完全不受 CPU 頻率、排程、溫度影響**，同樣輸入跑幾次都給同一個數字，所以它是雜訊環境下唯一穩定的比較基準 |
| `D refs` | 資料存取次數（讀 + 寫） | 搭配 miss 看比率 |
| `D1 misses` | L1 資料 cache 沒命中 | 要去 L2/L3 拿，貴 |
| `LL misses` | 最後一層 cache 也沒命中 | 真的要去 DRAM，最貴 |

> [!warning] `I refs` 不等於時間
> 兩段指令數相同的程式可以差好幾倍：一條等 DRAM 的 load 抵得上上百條算術指令，而現代 CPU 又會亂序執行、同時處理多筆記憶體請求。**`I refs` 回答「做了多少事」，miss 回答「等了多久」，時間是兩者的混合。** 三個都要看才拼得出全貌。

### 數據

隨機圖（BFS 跑 3 次）：

| | `I refs` | `D1 misses` | `LL misses` |
|---|---|---|---|
| `queue` | 92,205,929 | 7,385,918 | 9,048 |
| `cur/next` | 88,411,804 | 7,181,305 | 23,668 |

鏈狀圖：

| | `I refs` | `D1 misses` | `LL misses` |
|---|---|---|---|
| `queue` | 174,387,811 | 937,671 | 96,339 |
| `cur/next` | 204,001,251 | 937,613 | 96,259 |

### 判讀

照 [[Benchmarking-Toolkit]] 第三節的分離規則：

- **隨機圖：`I refs` 少 4%，miss 幾乎相同（差 2.8%）。** 少掉的 4% 是 `front()` / `pop()` 的簿記加上省掉的那次 `dist[u]` 讀取——但關鍵在 **738 萬次 D1 miss 兩邊一樣**。那些 miss 全部來自 `dist[v]` 的隨機存取，跟裝節點的容器毫無關係。**一個容器選擇改變不了的成本，佔了絕大部分執行時間**，這就是為什麼第二節的時間量不出差異：4% 的指令差躲在 738 萬次 miss 的陰影裡。
- **鏈狀圖：miss 幾乎一模一樣（差 58 次），`I refs` 多 17%。** 教科書等級的乾淨分離——差距 100% 是指令成本。100 萬層，每層都要付一次 `next.clear()`、一次 `swap()`、一次外層迴圈的起手與收尾。時間上的 0.73x 就是它。
- 兩張表的 `LL misses` 都是小數字（相對 D1 而言），代表工作集大致還壓在 L3 裡；真正的瓶頸是 L1 → L2/L3 這一段。

> [!tip] 這組數據同時解釋了 0102 為什麼會有 20%
> 那題的樹只有 2000 個節點——**整棵樹連同 `dist` 全部塞得進 32 KB 的 L1**，一次 miss 都不用付。沒有 miss 可以掩蓋，容器的指令成本就從 4% 的雜魚變成主角。**同一個優化在小資料上顯著、在大資料上消失**，是 cache 主導型工作負載的典型簽名。

## 四、記憶體高水位：hook `operator new`

`/usr/bin/time -v` 的 Max RSS 在這裡不夠用，兩個理由：它包含了圖本身（幾十 MB，把容器那幾 MB 淹掉），而且**釋放的記憶體不一定會還給作業系統**，RSS 只升不降。要看的是「容器實際持有的 live bytes 的高水位」，那得自己記帳：

```cpp
static size_t live = 0, peak = 0;
static unordered_map<void*, size_t>* sizes = nullptr;
static bool tracking = false;

void* operator new(size_t sz) {
  void* p = malloc(sz);
  if (!p) throw bad_alloc();
  if (tracking) {
    tracking = false;                    // ← 關鍵：擋掉重入
    (*sizes)[p] = sz;
    live += sz;
    peak = max(peak, live);
    tracking = true;
  }
  return p;
}
void operator delete(void* p) noexcept {
  if (tracking && p) {
    tracking = false;
    auto it = sizes->find(p);
    if (it != sizes->end()) { live -= it->second; sizes->erase(it); }
    tracking = true;
  }
  free(p);
}
void operator delete(void* p, size_t) noexcept { operator delete(p); }   // sized delete
```

> [!warning] 三個踩得到的坑
> 1. **重入**：記帳用的 `unordered_map` 自己也會呼叫 `operator new`，不擋就是無限遞迴。那個 `tracking` 旗標是必要的。
> 2. **sized delete**：C++14 起編譯器多半呼叫 `operator delete(void*, size_t)` 這個多載，只覆寫單參數版會漏掉大量釋放，`live` 一路虛高。兩個都要寫。
> 3. **測量區間要圈乾淨**：建圖那段要在 `tracking = true` 之前完成，否則圖的幾十 MB 會蓋掉一切。

### 數據（BFS 期間的 live bytes 高水位）

| 圖形 | `queue` | `cur/next` | 倍數 |
|---|---|---|---|
| 星形 n=1M（單層 100 萬） | 4066.48 KB | 6144.00 KB | 1.51x |
| **寬窄交替（20000 / 50 反覆 40 層）** | **81.48 KB** | **192.00 KB** | **2.36x** |
| 網格 1000×1000 | 4.80 KB | 10.00 KB | 2.08x |
| 鏈狀 n=1M | 1.06 KB | 0.01 KB | 0.01x |

（「寬窄交替」是為了這節多做的第五種圖：層寬在 20000 和 50 之間反覆跳，專門逼出高水位效應。）

### 判讀

`cur` / `next` 吃更多記憶體有兩個**互相疊加**的原因：

1. **`vector` 是倍增配置**，實際容量通常是需求的 1–2 倍（星形那列 100 萬個 `int` 只需要 4 MB，卻配了 6 MB）。
2. **`clear()` 不縮容**，而 `swap()` 讓兩個緩衝區輪流當主角——所以**兩個都會停在「歷史最寬那一層」的高水位**，不會因為當前層很窄就釋放。寬窄交替那列的 2.36x 就是這個效應：`deque` 的峰值貼著當下真正需要的 81 KB，兩個 `vector` 卻各自停在 96 KB。

反過來鏈狀圖那列 `cur` / `next` 幾乎不花錢（每層 1 個節點，兩個 vector 各佔幾個 byte），而 `deque` 最少也要配一整個 512 byte 的塊——**它是「起步價高、之後按需付費」，vector 則是「起步價低、但會囤貨」。**

## 五、結論與決策規則

**預設用 `queue`。** 三種寬圖上時間量不出差異（差距小於離散度），鏈狀圖上它明確勝出 37%，記憶體峰值穩定省 1.5–2.4x，而且沒有「忘記 `next.clear()`」這個踩雷點。

**改用 `cur` / `next` 的理由是表達力，不是速度**：

- 要按層輸出或聚合（[[0102-Binary-Tree-Level-Order-Traversal]]、0103、0107、0199、0637）
- 要免費的層號當距離，不想在佇列裡塞 `pair<node, dist>`
- 要對整層做整體操作：排序、去重、統計、選較小的一側擴展（雙向 BFS）、平行處理——所有平行 BFS 都是 frontier 陣列而非佇列，這是它真正的主場
- 資料小到全進 L1（LeetCode 的樹題正是這個規模），順帶賺到的常數才看得見

**根本不能換的場合**（界線是「下一批 ≠ 下一層」）：0-1 BFS（`deque` 要 `push_front` 插隊）、Dijkstra（出隊順序由距離決定）、SPFA（節點會重複入隊）。詳見 [[Shortest-Path-Algorithms]]。

> [!important] 方法論的收穫比結論本身有用
> 1. **時間量不出來 ≠ 沒有差異**，也 ≠ 有差異。要先看離散度才知道自己有沒有結論；量不出來時改用 `I refs` 這種不受頻率影響的指標。
> 2. **`I refs` 差 4% 卻完全反映不到時間上**，是因為 miss 才是主導項。**優化前先確認自己在優化的是主導項**——這裡真正該動的是 `dist[v]` 的存取成本（更小的 visited 表示法、節點重新編號），不是換容器。
> 3. **極端形狀的輸入是放大鏡**：鏈狀圖把每層固定開銷放大 100 萬倍，才讓一個在真實圖上看不見的成本現出原形。

## Related Problems

[[Benchmarking-Toolkit]] — 方法總表。這篇是它的完整案例，並補上 callgrind 圈區間與 `operator new` 記帳兩個手法
[[Micro-Optimization-Myths]] — 「換容器打不贏 cache miss」是那篇的典型條目，這裡是它在 BFS 上的實例
[[Graph-Traversal-and-Connectivity]] — BFS 骨架本身，以及 `cur` / `next` 的適用界線
[[Shortest-Path-Algorithms]] — 0-1 BFS 與 Dijkstra，也就是「下一批 ≠ 下一層」而不能套用的那些
[[0102-Binary-Tree-Level-Order-Traversal]] — 這場實驗的起點，那題的樹小到全進 L1，是常數優勢唯一看得見的規模
