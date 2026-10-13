---
tags:
  - sorting
  - quickselect
  - divide-and-conquer
  - stl
  - pattern
dg-publish: true
---

## 這篇在解決什麼

quicksort 大家都「會」，但面試現場手寫常出三種事：切分的不變量寫錯、固定 pivot 被已排序或全相同的測資打成 O(n²)、quickselect 的區間縮法搞混。另一邊是 STL：`sort`、`nth_element`、`partial_sort` 各自是哪個演算法、複雜度差在哪、什麼時候該選哪個。這篇把切分寫法、pivot 策略、quickselect 推導與 STL 對應放在一起。

## 一、切分（partition）的三種寫法

切分是 quicksort 唯一的核心動作：挑一個 pivot，把區間重排成「比它小的在左、比它大的在右」。三種寫法的差別在**不變量**，寫錯不變量就是 off-by-one 的來源。

### Lomuto — 最好記，但怕重複值

pivot 放尾端，單指標 `i` 維護「`[lo, i)` 全 < pivot」，掃完把 pivot 換到 `i`，pivot 就在最終位置。

```txt
[lo, i)     [i, j)      j → 掃描中     hi = pivot
 < pivot    >= pivot
```

```cpp
int lomuto(vector<int>& a, int lo, int hi) {
  int pivot = a[hi], i = lo;  // [lo, i) < pivot，[i, j) >= pivot
  for (int j = lo; j < hi; ++j) {
    if (a[j] < pivot) swap(a[i++], a[j]);
  }
  swap(a[i], a[hi]);
  return i;  // pivot 的最終位置，遞迴 [lo, i-1] 與 [i+1, hi]
}
```

> [!warning] 全相同的輸入會讓 Lomuto 退化成 O(n²)
> 所有元素都 `>= pivot`，`i` 完全不動，每輪只切掉 pivot 自己一格。n = 20000 全相同實測 51 ms，隨機只要 1 ms。LeetCode 的排序類題目一定有這組測資。

### Hoare — 雙指標向中間夾，天生耐重複值

pivot 取中間值（不是位置），`i` 從左找第一個 ≥ pivot、`j` 從右找第一個 ≤ pivot，交換後繼續，交錯時回傳 `j`。回傳後 `[lo, j]` 全 ≤ pivot、`[j+1, hi]` 全 ≥ pivot，但 **pivot 不一定停在 `j`**，所以遞迴是 `[lo, j]` 與 `[j+1, hi]`，不能像 Lomuto 那樣跳過 `j`。

```cpp
int hoare(vector<int>& a, int lo, int hi) {
  int pivot = a[lo + (hi - lo) / 2], i = lo - 1, j = hi + 1;
  while (true) {
    do ++i; while (a[i] < pivot);
    do --j; while (a[j] > pivot);
    if (i >= j) return j;
    swap(a[i], a[j]);
  }
}
// 遞迴：qs(a, lo, j); qs(a, j + 1, hi);
```

相等元素會被兩邊都停下來然後交換，所以全相同的輸入反而切得最平均，n = 20000 全相同 0.19 ms。比較次數比 Lomuto 少約三分之一，`std::sort` 用的就是 Hoare 式切分。

### 三路切分（Dutch National Flag） — 等於 pivot 的一次收掉

把區間切成三段：`[lo, lt)` < pivot、`[lt, i)` == pivot、`(gt, hi]` > pivot，`[i, gt]` 是還沒看的。等於 pivot 的整段一次定位，之後不用再碰，重複值越多越划算。

```txt
[lo, lt)     [lt, i)      [i, gt]        (gt, hi]
 < pivot     == pivot     未處理          > pivot

a[i] < pivot → swap(a[i], a[lt])，i++、lt++
a[i] > pivot → swap(a[i], a[gt])，gt--（i 不動，換過來的還沒看）
a[i] == pivot → i++
```

```cpp
pair<int, int> threeWay(vector<int>& a, int lo, int hi, int pivot) {
  int lt = lo, i = lo, gt = hi;
  while (i <= gt) {
    if (a[i] < pivot) {
      swap(a[i++], a[lt++]);
    } else if (a[i] > pivot) {
      swap(a[i], a[gt--]);
    } else {
      ++i;
    }
  }
  return {lt, gt};  // 等於 pivot 的區段是 [lt, gt]
}
```

這就是 [[0075-Sort-Colors]] 的解法，三種顏色就是 < = > 三段。

## 二、pivot 怎麼挑

| 策略 | 已排序輸入 | 全相同輸入 | 備註 |
| --- | --- | --- | --- |
| 固定首／尾 | O(n²) | Lomuto O(n²) | 教科書寫法，實戰必死 |
| 中間位置 | OK | 看切分 | 可被刻意構造打死，面試夠用 |
| median-of-3 | OK | 看切分 | `std::sort` 用的，首中尾取中位數 |
| 隨機 | 期望 O(n log n) | 看切分 | 沒有任何固定輸入能穩定打死它 |

兩個防線缺一不可：**pivot 策略**對付已排序（或刻意構造的）輸入，**切分方式**對付重複值。「隨機 pivot + 三路切分」兩個都顧到，是手寫時的推薦組合。n = 20000 實測（ms）：

```txt
                   random   sorted   allsame
Lomuto 固定尾端      1.16   105.81    50.84
Hoare 中間位置       1.34     0.18     0.19
三路 + 隨機          1.58     0.70     0.01
std::sort            0.97     0.12     0.10
```

> [!note]
> 隨機數用 `mt19937 rng(random_device{}());` 配 `uniform_int_distribution<int>(lo, hi)(rng)`，不要用 `rand() % (hi - lo + 1)`，後者在 LeetCode 的 MSVC／glibc 上品質參差，而且 `% 0` 的邊界要自己防。

## 三、quicksort 模板

```cpp
// Time: O(n log n) 平均，O(n²) 最壞（隨機 pivot 下機率可忽略）
// Space: O(log n) 遞迴棧
mt19937 rng(random_device{}());
void quicksort(vector<int>& a, int lo, int hi) {
  while (lo < hi) {
    int pivot = a[uniform_int_distribution<int>(lo, hi)(rng)];
    auto [lt, gt] = threeWay(a, lo, hi, pivot);
    if (lt - lo < hi - gt) {       // 小邊遞迴、大邊留在迴圈
      quicksort(a, lo, lt - 1);
      lo = gt + 1;
    } else {
      quicksort(a, gt + 1, hi);
      hi = lt - 1;
    }
  }
}
```

> [!tip] 小邊遞迴、大邊迴圈，棧深保證 O(log n)
> 每次遞迴只進較小的那一半，它最多是區間的一半，所以遞迴深度 ≤ log₂ n，即使切分很不平均也不會爆棧。兩邊都遞迴的寫法在退化時棧深是 O(n)。

quicksort **不穩定**（相等元素的相對順序會變），要穩定用 merge sort（`stable_sort`）。題目：[[0912-Sort-an-Array]]。

## 四、quickselect：只往一邊走

要「第 k 小」不需要全部排好，只要把第 k 小的放到位置 k、左邊全 ≤ 它、右邊全 ≥ 它。切分一次後看 k 落在哪段，**只往那段繼續，另外兩段直接丟掉**。

```cpp
// Time: O(n) 平均，O(n²) 最壞
// Space: O(1)
int quickselect(vector<int>& a, int k) {  // 升序第 k 個，0-based
  int lo = 0, hi = ssize(a) - 1;
  while (true) {
    int pivot = a[uniform_int_distribution<int>(lo, hi)(rng)];
    auto [lt, gt] = threeWay(a, lo, hi, pivot);
    if (k < lt) {
      hi = lt - 1;
    } else if (k > gt) {
      lo = gt + 1;
    } else {
      return pivot;  // k 落在 == pivot 的區段
    }
  }
}
```

> [!important] 為什麼是 O(n)
> 每輪切分花 O(區間長)，之後只留一邊，期望長度減半：n + n／2 + n／4 + … = 2n。quicksort 兩邊都要處理，每層加起來都是 n、共 log n 層，才是 O(n log n)。

> [!note] 第 k 大怎麼換算
> 第 k 大（1-based）= 升序的第 n − k 個（0-based）。或者把比較方向反過來、找降序的第 k − 1 個，[[0215-Kth-Largest-Element-in-an-Array]] 用的是後者。三路切分讓這題「全相同」的測資一輪就收：`k` 一定落在 `[lt, gt]` 直接回傳。

## 五、STL 對應

| 函式 | 演算法 | 複雜度 | 結果 |
| --- | --- | --- | --- |
| `sort` | introsort：Hoare 式 quicksort，遞迴深度超過 2·log n 改 heapsort，小區間（16）改 insertion sort | O(n log n) 保證 | 全部有序，不穩定 |
| `stable_sort` | merge sort（有額外記憶體時） | O(n log n) | 全部有序，穩定 |
| `nth_element(first, nth, last)` | introselect：quickselect，退化時改 heapselect | O(n) 平均，最壞 O(n log n) | `*nth` 是排序後該有的值，左邊全 ≤、右邊全 ≥，兩邊內部無序 |
| `partial_sort(first, mid, last)` | heapselect：對 `[first, mid)` 建 max-heap，掃剩下的比堆頂小就換 | O(n log k) | 前 k 個有序，其餘無序 |
| `partition` ／ `stable_partition` | 單趟切分 | O(n) | 依 predicate 分兩段 |
| `make_heap` ＋ `sort_heap` | heapsort | O(n log n) | 全部有序，不穩定 |

N = 10⁶ 隨機 `int` 實測（ms）：

```txt
k          sort    nth_element   partial_sort   手寫 quickselect
10         65.0        6.5           0.4              6.0
1000       65.4        6.6           0.9              4.1
100000     64.9        8.2          36.4              3.0
```

> [!tip] 前 k 名該用誰
> - k 很小：`partial_sort` 最快，它本質就是「大小 k 的 heap、先比堆頂」，而且結果有序，見 [[Heap-and-Priority-Queue]]。
> - k 中到大：`nth_element`，O(n) 跟 k 幾乎無關。
> - 資料是串流：只能 heap。
> - 值域很小：桶計數，確定性 O(n + 值域)。

> [!warning] `nth_element` 之後不要假設左邊有序
> 它只保證 `*nth` 定位、左邊全 ≤、右邊全 ≥。要「前 k 小且有序」得再對 `[first, nth)` 做 `sort`，或直接用 `partial_sort`。另外 comparator 一樣要是嚴格弱序，`<=` 會讓 introsort 的哨兵失效直接越界，見 [[STL-Pitfalls]]。

## Related Problems

- [[0215-Kth-Largest-Element-in-an-Array]] — quickselect 的原型題，手寫版與 `nth_element` 並列
- [[0973-K-Closest-Points-to-Origin]] — `nth_element` 對 pair 陣列選前 k 小，含 cache 考量
- [[0912-Sort-an-Array]] — 手寫 quicksort 的練習場，全相同測資會打死 Lomuto
- [[0075-Sort-Colors]] — 三路切分本體，三種顏色就是 < = > 三段
- [[0347-Top-K-Frequent-Elements]] — 對頻率做 quickselect 或桶排序
- [[0324-Wiggle-Sort-II]] — 先 `nth_element` 找中位數再三路切分的進階用法
- [[Heap-and-Priority-Queue]] — k 小或串流時的另一條路
- [[STL-Pitfalls]] — comparator 嚴格弱序，`<=` 讓 introsort 越界
