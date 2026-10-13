---
leetcode-id: 215
difficulty: medium
tags:
  - heap
  - grind-169
  - neetcode-150
memo: 大小 k 的 min-heap 堆頂即第 k 大，比堆頂大才進堆；題目暗示不排序的正解是 quickselect，手寫要隨機 pivot 加三路切分防全相同測資
dg-publish: true
---

## Problem Description

Given an integer array `nums` and an integer `k`, return the `kth` largest element in the array.

Note that it is the `kth` largest element in the sorted order, not the `kth` distinct element.

Can you solve it without sorting?

## Solution

「第 k 大」不需要把整個陣列排好。兩條路：**維護一個大小為 k 的 min-heap**裝目前最大的 k 個數，堆頂就是這 k 個裡最小的，也就是第 k 大，新數只要比堆頂大就把堆頂踢掉換它進來；或者**只把第 k 大的那個數放到它排序後該在的位置**，左右兩邊各自是「比它大」與「比它小」但內部不必有序，這正是 quickselect（`nth_element`）做的事，平均 O(n)。題目最後那句「Can you solve it without sorting?」就是在暗示後者。

> [!important]
> 「前 k 大」用 **min-heap**、「前 k 小」用 **max-heap**：堆頂永遠是候選集合裡「最該被淘汰」的那個，掃完整個陣列後堆頂就是第 k 名。

### 方法一：大小 k 的 min-heap — O(n log k)／O(k)

```cpp
// Time: O(n log k)
// Space: O(k)
class Solution {
 public:
  int findKthLargest(vector<int>& nums, int k) {
    priority_queue<int, vector<int>, greater<>> pq;  // 堆頂是候選裡最小的
    for (int n : nums) {
      if ((int)pq.size() < k) {
        pq.push(n);
      } else if (n > pq.top()) {  // 比堆頂大才值得換
        pq.pop();
        pq.push(n);
      }
    }
    return pq.top();
  }
};
```

> [!note]
> 更常見的寫法是每個數都先 `push`，再 `if (pq.size() > k) pq.pop()`，正確但每個數都付兩次 O(log k) 的 sift。隨機資料下第 i 個數擠得進前 k 名的機率只有 k／i，期望插入次數約 k·ln(n／k)，所以**先跟堆頂比、贏了才換**，絕大多數數一次比較就淘汰。n = 10⁵、k = 10 實測從 0.69 ms 降到 0.06 ms。另外 `pq.size()` 是 `size_t`，直接跟 `int k` 比會有 signed／unsigned 警告，轉型一下。

> [!tip] CTAD 寫法：前 k 個直接建堆
> C++17 起 `priority_queue` 有推導指引，給 iterator 對加 comparator 就能推出 `priority_queue<int, vector<int>, greater<>>`，三個模板參數都省掉。順便用建構子對前 k 個做 O(k) heapify，之後只掃剩下的，迴圈裡不用再判斷 `size() < k`。複雜度不變。

```cpp
// Time: O(n log k)
// Space: O(k)
class Solution {
 public:
  int findKthLargest(vector<int>& nums, int k) {
    priority_queue pq(nums.begin(), nums.begin() + k, greater<>{});  // 前 k 個建堆，O(k)
    for (int i = k; i < ssize(nums); ++i) {
      if (nums[i] > pq.top()) {
        pq.pop();
        pq.push(nums[i]);
      }
    }
    return pq.top();
  }
};
```

> [!note]
> C++23 規格裡有 `priority_queue pq(from_range, nums, greater<>{})`，但 libstdc++ 14 還沒實作，g++ 14 編不過。其他宣告寫法與 heap 的五種骨架見 [[Heap-and-Priority-Queue]]。

### 方法二：nth_element（quickselect） — 平均 O(n)／O(1)

```cpp
// Time: O(n) 平均
// Space: O(1)
class Solution {
 public:
  int findKthLargest(vector<int>& nums, int k) {
    ranges::nth_element(nums, nums.end() - k);  // 第 k 大 = 升序的倒數第 k 個
    return *(nums.end() - k);
  }
};
```

`nth_element` 保證 `nums.end() - k` 位置上是排序後該有的值，其左邊全 ≤ 它、右邊全 ≥ 它，但左右內部不排序。libstdc++ 用 introselect：quickselect 遞迴太深時改 heapselect 保底 O(n log n)，所以不怕惡意測資。

### 方法三：手寫 quickselect — 平均 O(n)／O(1)

面試常不准用 `nth_element`，要能手寫。每輪隨機挑 pivot 做**三路切分**，把區間分成「大於 pivot」「等於 pivot」「小於 pivot」三段，看第 k 大落在哪段就只往那段繼續，另外兩段直接丟掉。

```txt
目標：由大到小排的第 target = k-1 位

  [lo, lt)      [lt, i)      (gt, hi]
  > pivot      == pivot      < pivot      ← i 向右掃，未知區是 [i, gt]

  target < lt   → 答案在左段，hi = lt - 1
  target > gt   → 答案在右段，lo = gt + 1
  否則          → 答案就是 pivot
```

```cpp
// Time: O(n) 平均，O(n²) 最壞
// Space: O(1)
class Solution {
 public:
  int findKthLargest(vector<int>& nums, int k) {
    mt19937 rng(random_device{}());
    int lo = 0, hi = ssize(nums) - 1, target = k - 1;  // 由大到小排的第 target 位
    while (true) {
      int pivot = nums[uniform_int_distribution<int>(lo, hi)(rng)];
      int lt = lo, i = lo, gt = hi;  // [lo,lt) > pivot，[lt,i) == pivot，(gt,hi] < pivot
      while (i <= gt) {
        if (nums[i] > pivot) {
          swap(nums[i++], nums[lt++]);
        } else if (nums[i] < pivot) {
          swap(nums[i], nums[gt--]);
        } else {
          ++i;
        }
      }
      if (target < lt) {
        hi = lt - 1;
      } else if (target > gt) {
        lo = gt + 1;
      } else {
        return pivot;
      }
    }
  }
};
```

> [!warning]
> LeetCode 這題有「全部相同」和「已排序」的測資，**固定 pivot 的 Lomuto 切分會退化成 O(n²) 直接 TLE**。兩個防線缺一不可：隨機 pivot 對付已排序輸入；三路切分對付大量重複值，等於 pivot 的那段一次收掉、不會每輪只縮一格。

> [!tip]
> 每輪切分完只往一邊走，期望工作量是 n + n／2 + n／4 + … = 2n，這就是 quickselect 平均 O(n) 的來源；quicksort 兩邊都要遞迴才是 O(n log n)。

### 方法四：counting sort — O(n + 值域)／O(值域)

值域只有 [-10⁴, 10⁴]，直接數每個值出現幾次，從大往小扣 k，扣到 ≤ 0 的那個值就是答案。這是唯一**確定性** O(n) 的做法，但只在值域小時成立。

```cpp
// Time: O(n + 2·10⁴)
// Space: O(2·10⁴)
class Solution {
 public:
  int findKthLargest(vector<int>& nums, int k) {
    array<int, 20001> cnt{};
    for (int n : nums) {
      ++cnt[n + 10000];
    }
    for (int v = 20000;; --v) {
      k -= cnt[v];
      if (k <= 0) {
        return v - 10000;
      }
    }
  }
};
```

n = 10⁵ 的實測（wall clock，5 次取最佳，`-O2`，單位 ms）：

```txt
                   heap push再pop   heap 先比堆頂   nth_element   手寫 quickselect   counting sort
random k=10            0.690           0.063          0.684           0.406             0.058
random k=1000          1.880           0.374          0.708           0.316             0.059
random k=50000         4.675           4.146          0.838           1.033             0.065
allsame k=50000        1.089           0.188          0.111           0.103             0.063
```

> [!note]
> k ≪ n 時「先比堆頂」的 heap 幾乎每個數只做一次比較，跟 O(n) 的做法打平；k 接近 n／2 時 heap 退化成 O(n log n)，quickselect 才明顯勝出。heap 另一個優點是**串流友善**：資料一筆一筆來也能維護（見 [[0703-Kth-Largest-Element-in-a-Stream]]），quickselect 得先拿到全部。

## Related Problems

- [[0703-Kth-Largest-Element-in-a-Stream]] — 同一個大小 k 的 min-heap，但資料是串流，quickselect 用不上
- [[0973-K-Closest-Points-to-Origin]] — 前 k 小的版本，heap 與 nth_element 兩條路一樣，多了把鍵值抽出來再選的 cache 考量
- [[0347-Top-K-Frequent-Elements]] — 前 k 名的鍵值換成頻率，同樣可用 heap、quickselect 或桶排序
- [[0912-Sort-an-Array]] — 手寫 quicksort 的練習場，三路切分與隨機 pivot 的同一套防線
- [[Heap-and-Priority-Queue]] — `priority_queue` 宣告、CTAD 與「大小 k 的反向 heap」骨架
- [[Quicksort-and-Quickselect]] — 三種切分的不變量、pivot 策略與 `nth_element`／`partial_sort` 的選型
