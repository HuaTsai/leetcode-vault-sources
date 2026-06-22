---
leetcode-id: 338
difficulty: easy
tags:
  - bit-manipulation
  - grind-169
  - neetcode-150
memo: 逐數用 std::popcount 數 1-bit（輸入須無號）；替代技巧 n&(n-1) 清掉最右側的 1，反覆清到 0 的次數即位元數
dg-publish: true
---

## Problem Description

Given an integer `n`, return *an array* `ans` *of length* `n + 1` *such that for each* `i` (`0 <= i <= n`)*,* `ans[i]` *is the **number of*** `1`***'s** in the binary representation of* `i`.

## Solution

```cpp
// Time: O(n)
// Space: O(n)
vector<int> countBits(int n) {
  vector<int> ans;
  for (unsigned int i = 0; i <= n; ++i) {
    ans.push_back(std::popcount(i));
  }
  return ans;
}
```

如果不想使用 `std::popcount`，可以透過 `num & (num - 1)` 的技巧，該值有兩個特性

- 會移除 `num` 最右側的 1，所以可以持續移除直到成為 0，中間次數即為 1-bit 的數量
- 延伸自第一個特性，如果該值為 0，代表 `num` 本身為 2 的冪次，因此可以做快速判斷

但我建議是**統一使用 C++ 20 的 `std::popcount` 計算 1-bit 的數量，用 `std::has_single_bit` 檢查是否為 2 的冪次會比較容易**，唯一要注意的是輸入值必須要是無符號的整數

## Related Problems

[[1146-Counting-Bits]]
