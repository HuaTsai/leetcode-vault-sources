---
leetcode-id: 287
difficulty: medium
tags:
  - array
  - two-pointers
  - binary-search
  - bit-manipulation
  - pigeonhole
  - floyd
  - grind-169
  - neetcode-150
memo: 把 i → nums[i] 看成隱式鏈結串列，值恆落在 1 到 n、永遠指不回 index 0，所以必成 ρ 形而環的入口就是重複值；負號標記雖然更快卻改動了輸入、違反題目不得修改陣列的限制，且 Floyd 的相依載入鏈在大陣列上反而慢過 O(n log n) 的值域二分
dg-publish: true
---

## Problem Description

Given an array of integers `nums` containing `n + 1` integers where each integer is in the range `[1, n]` inclusive.

There is only one repeated number in `nums`, return this repeated number.

You must solve the problem without modifying the array `nums` and using only constant extra space.

Constraints:

- All the integers in `nums` appear only once except for precisely one integer which appears two or more times.


## Solution

核心觀念：`n + 1` 個數字塞進 `1..n` 的值域，由**鴿籠原理**保證一定有重複——難的從來不是「有沒有」，而是那兩條加碼限制：**不准改 `nums`、只能用 O(1) 空間**。拿掉它們這題就只剩 hash set 的變形；正是這兩條限制把它從陣列題推成了鏈結串列題。

關鍵轉換是把 `i → nums[i]` 看成一條**隱式鏈結串列**：index 是節點，值是 `next` 指標。

```txt
nums = [3, 1, 3, 4, 2]

index:   0    1    2    3    4
value:   3    1    3    4    2

把 index 當節點、值當 next 指標，從 index 0 出發：

  0 ──→ 3 ──→ 4 ──→ 2 ──┐
        ↑               │
        └───────────────┘

  尾巴：0 → 3          （長度 1，index 0 沒有任何入邊，所以一定在環外）
  環　：3 → 4 → 2 → 3  （長度 3）

  入口 3 有兩條入邊，分別來自 index 0 和 index 2
  而「能指向節點 v 的 index」就是「所有 nums[i] == v 的位置」
  → 兩條入邊 ⇔ 值 3 出現了兩次 ⇔ 答案就是 3
```

### 方法一：Floyd 判圈 — O(n)／O(1)（推薦）

完全符合題目兩條限制的線性解，也是這題真正想考的東西。第一階段快慢指針在環裡相遇，第二階段把一個指標拉回起點、兩者同速前進，交會處就是入環點。

```cpp
// Time: O(n)   兩階段各最多走一圈 ρ，隨機排列下實測平均約 0.8n 步
// Space: O(1)  只有兩個 int 當指標
class Solution {
 public:
  int findDuplicate(vector<int>& nums) {
    int slow = nums[0], fast = nums[0];
    do {
      slow = nums[slow];
      fast = nums[nums[fast]];
    } while (slow != fast);

    slow = nums[0];
    while (slow != fast) {
      slow = nums[slow];
      fast = nums[fast];
    }
    return slow;
  }
};
```

> [!important] 為什麼一定是 ρ 形，而且入口就是答案
> 值域是 `[1, n]`，所以 `nums[i]` 永遠不會是 `0`——**index 0 沒有任何入邊**。從 0 出發就絕不可能再回到 0，走出來的必定是「一條尾巴接一個環」的 ρ 形，而不是純圓；ρ 形才有「入口」可言。
> 再看入口：它是唯一有兩條以上入邊的節點，而能指向節點 `v` 的 index 恰好是所有滿足 `nums[i] == v` 的位置，兩條入邊即代表值 `v` 出現了兩次。**入環點的編號就是重複值本身**，不必再多做一步轉換。

> [!warning] 這裡必須是 `do-while`，不能寫成 `while`
> `slow` 與 `fast` 都從 `nums[0]` 起手，若寫成 `while (slow != fast)` 迴圈根本不會進去，直接把起點當成相遇點，第二階段立刻回傳 `nums[0]`。[[0141-Linked-List-Cycle]] 用 `while` 沒事是因為它只要一個 bool；這題需要拿到**相遇點**餵給第二階段，所以得用 `do-while` 強迫至少走一步。

> [!tip] 起點可以是 `nums[0]` 也可以是 `0`，但不能是別的 index
> 寫成 `slow = fast = 0` 再進同一個迴圈也對，兩者只差一個相位。關鍵是起點必須落在**尾巴上**（index 0 或它的後繼）：第二階段的距離等式 `尾巴長 = 相遇點繞回入口的距離` 是從「起點在環外」推出來的。若從某個恰好已在環內的 index 出發，尾巴長為 0，演算法會把起點自己當成入口回傳。

> [!note] 第二階段為什麼成立
> 設尾巴長 `μ`、環長 `λ`，相遇時 slow 走了 `k` 步且 `k` 是 `λ` 的倍數（因為 fast 剛好多繞整數圈）。此時再從起點走 `μ` 步會到入口；而相遇點往前 `μ` 步等於總共走了 `k + μ` 步，`k` 是整圈數所以位置與走 `μ` 步相同——兩者同時抵達入口。完整推導同 [[0142-Linked-List-Cycle-II]]。

### 方法二：值域二分 + 計數 — O(n log n)／O(1)

換個角度：陣列沒排序，不能對 index 二分，但可以**對答案的值域二分**。令 `cnt(mid)` 為陣列中 `≤ mid` 的元素個數。若沒有重複，`1..mid` 各出現一次，`cnt(mid) == mid`；重複值一旦 `≤ mid`，就會把 `cnt(mid)` 頂到 `> mid`。述詞「`cnt(mid) > mid`」對 `mid` 單調，可以二分。

```cpp
// Time: O(n log n)  值域二分 log n 輪，每輪掃一次陣列
// Space: O(1)       不改動 nums，符合題目限制
class Solution {
 public:
  int findDuplicate(vector<int>& nums) {
    int lo = 1, hi = nums.size() - 1;
    while (lo < hi) {
      int mid = lo + (hi - lo) / 2;
      int cnt = 0;
      for (int x : nums) {
        cnt += (x <= mid);
      }
      if (cnt > mid) {
        hi = mid;
      } else {
        lo = mid + 1;
      }
    }
    return lo;
  }
};
```

> [!tip] 這就是「找第一個讓述詞成立的值」模板
> 述詞是 `cnt(mid) > mid`，成立時收右界 `hi = mid`（保留 mid 這個候選），否則 `lo = mid + 1`。`lo == hi` 時區間收到唯一解，不必再額外檢查。模板細節見 [[Binary-Search-Templates]]。

> [!warning] `hi` 是 `nums.size() - 1` 而不是 `nums.size()`
> 二分的是**值域** `[1, n]`，而陣列長度是 `n + 1`，所以 `n == nums.size() - 1`。寫成 `nums.size()` 會多出一個不可能成立的候選值 `n + 1`；雖然這題因為述詞單調仍會收斂到正解，但一旦把它當模板抄去別題就會出事。

### 方法三：逐位計數 — O(n log n)／O(1)

鴿籠原理的另一種切法：不比對「值出現幾次」，改成逐個 bit 比對「這一位是 1 的數字有幾個」。基準是 `1..n` 的理論值，超出基準的那幾位就拼出答案。

```cpp
// Time: O(n log n)  每個 bit 掃一次陣列，共 log n 個 bit
// Space: O(1)       不改動 nums
class Solution {
 public:
  int findDuplicate(vector<int>& nums) {
    int n = nums.size() - 1, ans = 0;
    int bits = 32 - __builtin_clz(n);
    for (int b = 0; b < bits; ++b) {
      int base = 0, cnt = 0;
      for (int i = 0; i <= n; ++i) {
        if (i & (1 << b)) ++base;
        if (nums[i] & (1 << b)) ++cnt;
      }
      if (cnt > base) {
        ans |= 1 << b;
      }
    }
    return ans;
  }
};
```

> [!tip] 同一個 `i` 同時當 index 和「基準值」用
> 迴圈跑 `i = 0..n`：`nums[i]` 把 `i` 當 index 掃完整個陣列，`i & (1 << b)` 則把 `i` 當值域裡的數字算基準。因為值域是 `1..n` 而 `i == 0` 對任何 bit 都貢獻 0，兩個角色的範圍剛好對得起來，不必寫成兩個迴圈。

> [!important] 重複值出現超過兩次時，這招為什麼還是對的
> 設重複值出現 `k` 次（`k ≥ 2`），陣列共 `n + 1` 格，所以有 `k - 2` 個值缺席（不是 `k - 1`，別算錯）。對第 `b` 位，記缺席值中該位為 1 的個數為 `s`（`0 ≤ s ≤ k - 2`）：
> - 答案該位為 0：`cnt = base - s ≤ base`，不會誤判成 1。
> - 答案該位為 1：`cnt = base + (k - 1) - s ≥ base + (k - 1) - (k - 2) = base + 1 > base`，一定判得出來。
>
> 兩邊都是**嚴格**的，所以 `cnt > base` 恰好等價於「答案該位為 1」。

> [!note] `__builtin_clz(0)` 是未定義行為
> 這題保證 `n ≥ 1` 所以安全。真要偷懶，無腦跑滿 32 輪其實也對（超過 `n` 最高位的那些 bit，兩邊計數都是 0），代價是白花約 3 倍時間。

### 方法四：負號標記 — O(n)／O(1)，但**破壞輸入**

利用「值域 `1..n` 剛好可以當 index 用」：走到值 `v` 就把 `nums[v - 1]` 塗負，再遇到已經是負的就代表 `v` 來過第二次。正負號本身就是那張 visited 表，不用額外空間。

```cpp
// Time: O(n)   兩份重複值的位置隨機時，平均掃到約 2n/3 就撞上
// Space: O(1)  就地拿正負號當標記
class Solution {
 public:
  int findDuplicate(vector<int>& nums) {
    int n = nums.size();
    for (int i = 0; i < n; ++i) {
      int idx = abs(nums[i]) - 1;
      if (nums[idx] < 0) {
        return idx + 1;
      } else {
        nums[idx] *= -1;
      }
    }
    return -1;
  }
};
```

> [!warning] 這解法不符合題目要求——它改了 `nums`
> 題目明寫 *"without modifying the array `nums`"*。LeetCode 的 judge 不檢查陣列狀態，所以照樣 AC，但**這條限制就是本題的全部難度**：拿掉它，287 只是 [[0217-Contains-Duplicate]] 的變形。面試寫這個要先講清楚「我知道它違反限制，合規版本是 Floyd」，否則等於答非所問。

> [!tip] `abs()` 不能省
> 讀 `nums[i]` 時它可能早就被前面某一輪塗成負的了，直接拿去減一會算出負 index。`abs` 的作用是**還原原始值再當 index 用**——標記只存在符號位，數值本身要留著。

想留住這個寫法又不留下痕跡，可以掃完把動過的符號改回來（實測還原後陣列與原輸入完全一致）：

```cpp
// Time: O(n)   多掃一趟把符號改回來，常數約兩倍
// Space: O(1)
class Solution {
 public:
  int findDuplicate(vector<int>& nums) {
    int n = nums.size(), ans = -1, last = n;
    for (int i = 0; i < n; ++i) {
      int idx = abs(nums[i]) - 1;
      if (nums[idx] < 0) {
        ans = idx + 1;
        last = i;
        break;
      }
      nums[idx] *= -1;
    }
    for (int i = 0; i < last; ++i) {  // 只需還原 break 之前動過的那些
      int idx = abs(nums[i]) - 1;
      if (nums[idx] < 0) {
        nums[idx] *= -1;
      }
    }
    return ans;
  }
};
```

> [!note] 還原也不等於合規
> 過程中陣列仍然被改過：多執行緒共享、或簽名是 `const vector<int>&` 時一樣不能用。而且代價不小——見下方實測，n = 1e6 時從 1.5 ms 變 5.7 ms，剛好被值域二分追平。

> [!note] 實測：指令數最少的 Floyd 反而最慢
> `g++ -std=c++20 -O2`，隨機排列輸入，時間已扣掉複製陣列的成本；指令數與 cache miss 由 cachegrind 量測並扣掉建測資的 baseline。
>
> | n = 1e6（陣列 4 MB） | 指令數 | D1 read miss | 時間 |
> | --- | --- | --- | --- |
> | 負號標記 | 8.7 M | 0.61 M | **1.5 ms** |
> | 值域二分 | 140 M | 1.11 M | 5.7 ms |
> | 逐位計數 | — | — | 13.9 ms |
> | Floyd 判圈 | **4.3 M** | 2.40 M | **34.8 ms** |
>
> Floyd 的指令數只有二分的 1/32，時間卻慢了 6 倍。原因是 `slow = nums[slow]` 構成一條**相依載入鏈**——下一筆的位址要等上一筆 load 回來才算得出來，cache miss 只能一個接一個排隊付完整的記憶體延遲（34.8 ms ÷ 2.4 M miss ≈ 15 ns，正是一次 DRAM 往返）。二分是順序掃描，可預取又可向量化；負號標記是「順序讀 + 隨機寫」，寫入位址彼此獨立，亂序執行能同時吃下好幾筆 miss。**指令數不是速度，記憶體相依性才是。**
>
> 但在題目實際的上限 `n ≤ 1e5`（400 KB，塞得進 L2）：Floyd 0.47 ms、值域二分 0.48 ms、負號標記 0.08 ms——Floyd 與二分**完全打平**。「O(n) 必定勝過 O(n log n)」在這題不成立，複雜度只在陣列大到裝不進 cache 時才開始說話，而那時贏的還是二分。

> [!tip] Floyd 的步數是 Θ(n)，不是 O(√n)
> 隨機排列下實測平均走 `0.8n` 步，`n` 從 1e3 到 1e7 都是這個比例。O(√n) 是**隨機函數**的 ρ 長度（生日悖論那套），而這題的 `nums` 是一個排列加上一份重複值，環長期望是 Θ(n)。兩者別搞混。

## Related Problems

- [[0141-Linked-List-Cycle]] — Floyd 的原型，這題只是把「指標」換成「index → 值」的隱式串列
- [[0142-Linked-List-Cycle-II]] — 本題第二階段完全照搬那題的找入環點，距離等式的證明也在那邊
- [[0202-Happy-Number]] — 另一種隱式串列（數字迭代），同樣用 Floyd 判圈
- [[0041-First-Missing-Positive]] — 同樣吃「值域 `1..n` 可以當 index 用」這個性質，但它是就地交換而非塗符號
- [[0448-Find-All-Numbers-Disappeared-in-an-Array]] — 負號標記法的正宗題目，那題沒有禁止修改陣列所以可以放心用
- [[0217-Contains-Duplicate]] — 拿掉「不准改陣列 + O(1) 空間」之後，287 就退化成這題
- [[0268-Missing-Number]] — 鴿籠原理的對偶面，找的是缺席而非重複
- [[0875-Koko-Eating-Bananas]] — 同樣是「對答案的值域二分」而非對陣列二分
- [[Binary-Search-Templates]] — 方法二用的「找第一個成立的值」模板
- [[Micro-Optimization-Myths]] — 那篇談除法與分支預測，這題補上第三條軸：相依載入鏈造成的記憶體延遲
