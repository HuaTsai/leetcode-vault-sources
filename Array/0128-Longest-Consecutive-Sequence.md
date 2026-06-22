---
leetcode-id: 128
difficulty: medium
tags:
  - array
  - hash
  - union-find
  - grind-169
  - neetcode-150
memo: 標準解用 unordered_set，只從序列起點（n-1 不存在者）往上數達 O(n)；亦可 Union-Find，但務必只 union 存在的鄰居，否則 operator[] 會製造幽靈節點 0 把無關數字併在一起
dg-publish: true
---

## Problem Description

Given an unsorted array of integers `nums`, return the length of the longest consecutive elements sequence.

You must write an algorithm that runs in `O(n)` time.

A **_Longest Consecutive Sequence_** is the longest chain of numbers in a given dataset (like an array or tree) where each number is exactly one integer greater than the previous one.

## Solution

目標是在 O(n) 內找出最長的「連續整數」串。先把數字放進 hash 容器、用 O(1) 查詢「某個數在不在」，是兩種解法共同的基礎。

### 方法一：Hash Set — O(n) 標準解

把所有數丟進 `unordered_set`，然後**只從序列起點往上數**。所謂起點，就是 `n - 1` 不在集合裡的數。

> [!tip] O(n) 的關鍵：只從序列起點開始數
> 對每個數先看 `n - 1` 在不在——在的話它不是起點，直接跳過；只有起點才往 `n+1, n+2, …` 一路數。如此每個數最多被「起點判斷」與「往上數」各走訪一次，總計 O(n)。少了這個守衛、對每個數都往上數，就會退化成 O(n²)。

```cpp
// Time: O(n)
// Space: O(n)
class Solution {
 public:
  int longestConsecutive(vector<int>& nums) {
    unordered_set<int> s(nums.begin(), nums.end());
    int ans = 0;
    for (int n : s) {
      if (s.count(n - 1)) continue;   // 不是序列起點就跳過
      int cur = n, len = 1;
      while (s.count(cur + 1)) {       // 從起點往上數
        ++cur;
        ++len;
      }
      ans = max(ans, len);
    }
    return ans;
  }
};
```

### 方法二：Union-Find — 把相鄰數字併進同一集合

換個角度：把每個數字當節點，相鄰（差 1）的數字 union 在一起，最後最大的 component 大小就是答案。因為元素是任意整數（可能為負或很大），`parent`／`size` 用 `unordered_map` 而非陣列。

```cpp
// Time: O(n·α(n))（路徑壓縮 + union by size，近似 O(n)）
// Space: O(n)
struct UnionFind {
  unordered_map<int, int> parent, size;

  void add(int x) {
    if (parent.count(x)) return;            // 防重複數字重設結構
    parent[x] = x;
    size[x] = 1;
  }
  int find(int x) {
    if (parent[x] != x) parent[x] = find(parent[x]);
    return parent[x];
  }
  void unite(int x, int y) {
    int px = find(x), py = find(y);
    if (px == py) return;
    if (size[px] > size[py]) swap(px, py);  // union by size（可選，掛小到大）
    parent[px] = py;
    size[py] += size[px];
  }
};

class Solution {
 public:
  int longestConsecutive(vector<int>& nums) {
    UnionFind uf;
    for (int n : nums) uf.add(n);                       // 1. 先把所有數加入
    for (int n : nums) {
      if (uf.parent.count(n - 1)) uf.unite(n, n - 1);   // 2. 只 union 存在的鄰居
      if (uf.parent.count(n + 1)) uf.unite(n, n + 1);
    }
    int ans = 0;
    for (auto& [k, sz] : uf.size) ans = max(ans, sz);
    return ans;
  }
};
```

> [!warning] 陷阱：別對「不存在的鄰居」呼叫 find
> 最容易踩的雷是無條件 `unite(n, n - 1)`、`unite(n, n + 1)`。`find` 裡的 `parent[x]`（`operator[]`）在 key 不存在時會**自動插入並初始化為 0**，於是不存在的鄰居會 collapse 到「幽靈根 0」，把本來不相鄰的數字錯誤地併在一起——例如 `[1,3]` 會算成 2、`[10,20,30]` 算成 3。修正方式就是上面的 `if (uf.parent.count(n ± 1))`：**只在鄰居確實存在時才 union**。

兩個重點：

- **為什麼拆兩個迴圈**：先把所有數 `add` 進結構，再逐一檢查 `n-1`／`n+1` 是否存在、存在才 `unite`。這樣連邊前保證元素都已存在，不必去推理陣列順序（單迴圈 + 守衛其實也對，較晚處理的那個會反向接上，只是要多想一層）。
- **`size` 一身兼二職**：既是 union by size 的依據，也是 component 大小（即答案）。`size[py] += size[px]` 在有沒有那行 `swap` 的情況下都會在根節點正確累加；拿掉 `swap` 仍正確，只是退化到約 O(n log n)。

> [!note] 補充
> Union-Find 是完全正確的替代解，但常數與程式長度都比 hash set 高。**LC128 的公認標準解是方法一的 hash set O(n)**，面試時通常先想到它。

## Related Problems

[[0217-Contains-Duplicate]] — 同樣以 `unordered_set` 為核心的查詢／去重
[[0200-Number-of-Islands]] — Union-Find 連通分量的經典應用
[[0684-Redundant-Connection]] — 直接練 Union-Find 的 find／unite 與環偵測
