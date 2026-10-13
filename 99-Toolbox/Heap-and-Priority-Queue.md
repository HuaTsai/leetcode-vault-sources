---
tags:
  - heap
  - priority-queue
  - stl
  - pattern
dg-publish: true
---

## 這篇在解決什麼

heap 本身不難：一個能在 O(log n) 內「拿走最大（或最小）」和「丟一個進去」的容器。難的是三件事：

1. **`priority_queue` 的宣告**——預設是 max-heap、comparator 的方向跟直覺相反、想省掉三個模板參數有幾種寫法
2. **什麼題型該想到它**——「前 k 名」「k 路合併」「每次取最極端的兩個」「動態中位數」「要刪中間元素但 heap 不支援」，骨架各不相同
3. **什麼時候不該用**——k 接近 n、值域很小、要改值或刪中間，各有更好的工具

## 一、底層：陣列裝的完全二元樹

heap 是一棵**完全二元樹**，所以可以不用指標、直接平鋪在陣列裡：節點 `i` 的孩子是 `2i+1`、`2i+2`，父親是 `(i-1)/2`。max-heap 的不變量只有一條：**每個節點 ≥ 它的兩個孩子**，所以根一定是全場最大，但兄弟之間、不同子樹之間沒有任何順序。

```txt
陣列  [9, 7, 8, 3, 5, 6, 2]
樹            9
            /   \
           7     8
          / \   / \
         3   5 6   2

push(10)：放到尾端 → 一路跟父親比、比父親大就交換上浮（sift up），O(log n)
pop()   ：把尾端搬到根 → 跟較大的孩子比、比它小就交換下沉（sift down），O(log n)
```

> [!important] 建堆是 O(n)，不是 O(n log n)
> 把 n 個元素逐一 `push` 是 O(n log n)；但 `make_heap` 從最後一個非葉節點往前逐個 sift down，深層節點多但下沉距離短、淺層節點少但下沉距離長，加總是 O(n)。n = 4×10⁶ 實測 38 ms 對 75 ms。所以一開始就有全部資料時，用 `make_heap` 或帶容器的建構子，不要迴圈 `push`。

## 二、`priority_queue` 的宣告

```cpp
priority_queue<int> mx;                           // 預設 less<>，max-heap，top() 是最大
priority_queue<int, vector<int>, greater<>> mn;   // min-heap，top() 是最小
priority_queue<pair<int, int>> pr;                // pair 先比 first 再比 second，不用自訂 comparator
```

> [!warning] comparator 回答的是「誰排後面」，所以 `greater` 給出 min-heap
> `Compare(a, b)` 為 true 代表 a 的優先度**低於** b。`less` 讓小的排後面、大的先出來，是 max-heap；`greater` 反過來。自訂 lambda 時最容易寫反：想要小的先出來，要寫 `return a.val > b.val;`。寫 `>=` 則是違反嚴格弱序的 UB，見 [[STL-Pitfalls]]。

### 自訂比較

```cpp
// lambda：C++20 起無捕獲 lambda 可預設建構，不用再 pq(cmp)
auto cmp = [](const Task& a, const Task& b) { return a.pri > b.pri; };
priority_queue<Task, vector<Task>, decltype(cmp)> pq;

// 或讓編譯器生 operator<：一行 <=> 就能直接丟進 priority_queue / sort / set
struct Task {
  int pri;
  string name;
  auto operator<=>(const Task&) const = default;  // 依序比 pri、再比 name
};
priority_queue<Task> pq2;
```

### CTAD：省掉三個模板參數（C++17）

`priority_queue` 有推導指引，只要給得出 comparator 和容器型別，三個模板參數都能省：

```cpp
vector<int> v{5, 1, 4, 2, 3};
priority_queue pq1(v.begin(), v.end(), greater<>{});  // → priority_queue<int, vector<int>, greater<>>，O(n) 建堆
priority_queue pq2(greater<>{}, vector<int>{});       // 空的 min-heap
priority_queue pq3(less<>{}, std::move(v));           // 接管既有 vector，O(n) 建堆，不複製
```

第一種對「大小 k 的 heap」特別順手：`priority_queue pq(nums.begin(), nums.begin() + k, greater<>{});` 先把前 k 個 O(k) 建好，之後的迴圈就不用判斷 `size() < k`，見 [[0215-Kth-Largest-Element-in-an-Array]]。

> [!note]
> C++23 規格有 `priority_queue pq(from_range, nums, greater<>{})`，但 libstdc++ 14 還沒實作，g++ 14 編不過。`priority_queue` 沒有 `min_priority_queue` 這類別名，三參數版仍是正規寫法；自己 `using min_heap = priority_queue<int, vector<int>, greater<>>;` 是常見做法。

### 原地用底層演算法

`priority_queue` 只是 `make_heap`／`push_heap`／`pop_heap` 包了一層。直接對 `vector` 操作就不用另開容器，輸入本身就是 heap 的儲存體，見 [[1046-Last-Stone-Weight]]。

```cpp
ranges::make_heap(v);                    // O(n)，之後 v.front() 是最大
v.push_back(x); ranges::push_heap(v);    // O(log n)
ranges::pop_heap(v);                     // 堆頂搬到 v.back()，[begin, end-1) 仍是 heap
int top = v.back(); v.pop_back();
ranges::sort_heap(v);                    // 由 heap 得到升序，這就是 heapsort
```

## 三、五種骨架

### 骨架一：前 k 名 — 大小 k 的「反向」heap

要「前 k 大」就維護一個**大小 k 的 min-heap**：堆頂是候選裡最小的，也就是最該被淘汰的。新元素比堆頂大才換進來，掃完堆裡就是前 k 大、堆頂就是第 k 大。「前 k 小」則用 max-heap。O(n log k)、空間 O(k)。

```cpp
priority_queue<int, vector<int>, greater<>> pq;  // 前 k 大 → min-heap
for (int n : nums) {
  if ((int)pq.size() < k) {
    pq.push(n);
  } else if (n > pq.top()) {  // 比堆頂大才值得換
    pq.pop();
    pq.push(n);
  }
}
```

> [!tip] 先比堆頂再 push，不要 push 完再 pop
> 隨機資料下第 i 個元素擠進前 k 名的機率只有 k／i，期望只有 k·ln(n／k) 次插入，絕大多數元素一次比較就淘汰。「每個都 push、超過 k 就 pop」正確但每個元素都付兩次 sift，k ≪ n 時慢 5～10 倍，[[0973-K-Closest-Points-to-Origin]] 與 [[0215-Kth-Largest-Element-in-an-Array]] 都有實測。

題目：[[0215-Kth-Largest-Element-in-an-Array]]、[[0973-K-Closest-Points-to-Origin]]、[[0347-Top-K-Frequent-Elements]]、[[0703-Kth-Largest-Element-in-a-Stream]]（串流版，heap 的主場）。

### 骨架二：k 路合併 — 每路只放一個代表

k 條有序序列合併成一條：heap 裡**永遠只有每條序列各一個 head**，pop 最小的接到輸出，再把它的下一個 push 回去補位。堆大小恆為 k，所以是 O(N log k) 而不是 O(N log N)。

```cpp
using E = tuple<int, int, int>;  // {值, 第幾路, 路內索引}
priority_queue<E, vector<E>, greater<>> pq;
for (int i = 0; i < ssize(lists); ++i) {
  if (!lists[i].empty()) pq.emplace(lists[i][0], i, 0);
}
while (!pq.empty()) {
  auto [v, li, j] = pq.top();
  pq.pop();
  out.push_back(v);
  if (j + 1 < ssize(lists[li])) pq.emplace(lists[li][j + 1], li, j + 1);
}
```

題目：[[0023-Merge-k-Sorted-Lists]]、[[0378-Kth-Smallest-Element-in-a-Sorted-Matrix]]、[[0373-Find-K-Pairs-with-Smallest-Sums]]。

### 骨架三：模擬「每次取最極端的兩個」

規則是「拿最大的兩個做某件事、結果丟回去」，heap 就是這個流程的直譯。注意**兩個都 pop 完再 push 結果**，否則結果可能被當成第二個拿去用。題目：[[1046-Last-Stone-Weight]]、[[0767-Reorganize-String]]、[[0621-Task-Scheduler]]。

### 骨架四：lazy deletion — heap 不能刪中間，就讓過期的留著

heap 只能拿堆頂，不能刪指定元素。做法是**不刪**：元素存 {值，索引或版本}，取堆頂時先檢查它是否已過期，過期就丟、再看下一個。過期但不在堆頂的元素留在堆裡不影響正確性，只多佔空間。

```cpp
// 滑動視窗最大值：heap 版（O(n log n)，deque 版見 Monotonic-Stack）
priority_queue<pair<int, int>> pq;  // {值, 索引}
for (int i = 0; i < ssize(nums); ++i) {
  pq.emplace(nums[i], i);
  while (pq.top().second <= i - k) pq.pop();  // 堆頂過期才丟
  if (i >= k - 1) ans.push_back(pq.top().first);
}
```

Dijkstra 的「同一節點被 push 多次、pop 出來發現距離比已知大就跳過」是同一招，見 [[Shortest-Path-Algorithms]]。題目：[[0239-Sliding-Window-Maximum]]、[[0480-Sliding-Window-Median]]。

### 骨架五：兩個 heap 夾中位數

較小的一半放 max-heap `lo`、較大的一半放 min-heap `hi`，兩個堆頂就是中間兩個數。每次插入先進 `lo`、再把 `lo` 的最大擠到 `hi`，保證 `lo` 全 ≤ `hi` 全；最後把 size 平衡回 `lo.size() ≥ hi.size()`。

```cpp
priority_queue<int> lo;                          // 較小的一半，堆頂是其中最大
priority_queue<int, vector<int>, greater<>> hi;  // 較大的一半，堆頂是其中最小
void addNum(int x) {
  lo.push(x);
  hi.push(lo.top()); lo.pop();                                  // 讓 lo 全 ≤ hi 全
  if (hi.size() > lo.size()) { lo.push(hi.top()); hi.pop(); }  // 平衡
}
double findMedian() {
  return lo.size() > hi.size() ? lo.top() : (lo.top() + hi.top()) / 2.0;
}
```

題目：[[0295-Find-Median-from-Data-Stream]]、[[0502-IPO]]（兩個 heap 但一個是「解鎖」用）。

## 四、什麼時候不要用 heap

| 情境 | 更好的工具 | 原因 |
| --- | --- | --- |
| k 接近 n／2 的第 k 名 | `nth_element`（quickselect） | heap 退化成 O(n log n)，quickselect 平均 O(n)，見 [[Quicksort-and-Quickselect]] |
| 值域很小（如 ≤ 10⁴） | 桶計數 | O(n + 值域) 且確定性，[[1046-Last-Stone-Weight]]、[[0215-Kth-Largest-Element-in-an-Array]] 都有 |
| 要改值、刪中間、找特定元素 | `multiset`／`map`，或 `__gnu_pbds::priority_queue` | heap 沒有 `erase(x)`，lazy deletion 不夠用時換結構，[[0347-Top-K-Frequent-Elements]] 有 PBDS 示範 |
| 前 k 名還要排好序 | `partial_sort` | 它就是 heapselect，k 小時比 `nth_element` 更快 |
| 全部都要排序 | `sort` | 一次 heapsort 不會比 introsort 快 |
| 邊權只有 0／1 的最短路 | `deque`（0-1 BFS） | 距離只有兩種值，不需要 log，見 [[Shortest-Path-Algorithms]] |

## 五、常見陷阱

- **comparator 方向反**：`greater` 才是 min-heap；自訂 lambda 想小的先出要寫 `>`。
- **`>=` 不是嚴格弱序**：heap 演算法通常不會炸，但已是 UB，`sort` 用同一份會當場越界，見 [[STL-Pitfalls]]。
- **空 heap 的 `top()`／`pop()` 是 UB**，不會丟例外，先檢查 `empty()`。
- **`pq.size()` 是 `size_t`**：跟 `int k` 比會有 signed／unsigned 警告，`pq.size() > k - 1` 這類算式在 k = 0 時更會繞成極大值。轉型後再比。
- **heap 不穩定、不可迭代有序**：平手順序不保證；要「值相同保留輸入順序」就把索引一起放進 key。`priority_queue` 也沒有 `begin()`，想看全部內容只能 pop 光。
- **`pop_heap` 後的尾端要自己 `pop_back`**：它只搬不刪。

## Related Problems

- [[0215-Kth-Largest-Element-in-an-Array]] — 骨架一的原型，heap 與 quickselect 兩條路並列
- [[0973-K-Closest-Points-to-Origin]] — 前 k 小用 max-heap，先比堆頂的實測
- [[0703-Kth-Largest-Element-in-a-Stream]] — 串流版，heap 相對 quickselect 的真正優勢
- [[0347-Top-K-Frequent-Elements]] — 鍵值換成頻率，附 PBDS 可改值 heap 的示範
- [[0023-Merge-k-Sorted-Lists]] — 骨架二，每路一個代表、堆大小恆為 k
- [[1046-Last-Stone-Weight]] — 骨架三，附原地 `make_heap`／`pop_heap` 寫法
- [[0239-Sliding-Window-Maximum]] — 骨架四 lazy deletion 的 heap 解，對照單調 deque
- [[0295-Find-Median-from-Data-Stream]] — 骨架五，兩個 heap 夾中位數
- [[Quicksort-and-Quickselect]] — k 大時換這條路
- [[STL-Pitfalls]] — comparator 嚴格弱序與 `<=>`
