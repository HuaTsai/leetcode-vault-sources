---
leetcode-id: 424
difficulty: medium
tags:
  - sliding-window
  - hash
  - string
  - grind-169
  - neetcode-150
memo: 收縮條件是「窗長減去出現最多的字元數大於 k」；最易錯在縮窗後沒重算長度就更新答案，等於把不合法的窗記進去，答案只會偏大；進階版讓窗永不縮小、maxcnt 只增不減即可 O(n)，因為過期的 maxcnt 只會出現在平移期間、不影響決策
dg-publish: true
---

## Problem Description

You are given a string `s` and an integer `k`. You can choose any character of the string and change it to any other uppercase English character. You can perform this operation at most `k` times.

Return the length of the longest substring containing the same letter you can get after performing the above operations.

Constraints:

- `s` consists of only uppercase English letters.

## Solution

核心觀念：一個窗口 `s[l..r]` 合法，代表**把窗口裡「非多數字元」的部分全部改掉，次數不超過 `k`**。設窗口內出現最多的字元有 `maxcnt` 個，需要的替換次數就是 `窗長 − maxcnt`。所以判斷式是：

```txt
窗長 − maxcnt > k   →   不合法
```

注意**不需要枚舉「最後統一成哪個字元」**——變成多數字元一定是最省的選擇，所以只要盯著 `maxcnt` 這一個數字。

```txt
s = A A B A B B A ，k = 1
    0 1 2 3 4 5 6

 r  窗口        窗長  maxcnt  要改幾次  合法？
 0  [0,0] A       1     1        0      ✓
 1  [0,1] AA      2     2        0      ✓
 2  [0,2] AAB     3     2        1      ✓
 3  [0,3] AABA    4     3        1      ✓  ← 答案 4
 4  [0,4] AABAB   5     3        2      ✗  超過 k=1
```

> [!important] 這是 [[0003-Longest-Substring-Without-Repeating-Characters]] 的同一個骨架
>
> ```txt
> for (r = 0; r < n; ++r) {
>   加入 s[r]
>   while (窗口不合法) { 移除 s[l]; ++l; }
>   更新答案
> }
> ```
>
> 換題只換那一行判斷：0003 是「有重複」，本題是「窗長 − maxcnt > k」。**先把骨架寫對，再談優化。**

### 方法一：骨架版，縮到合法為止 — O(26n)／O(1)（推薦）

```cpp
// Time: O(26n)（每輪重算一次 26 格的最大值；l 只前進不後退，縮窗均攤 O(n)）
// Space: O(1)
class Solution {
 public:
  int characterReplacement(string s, int k) {
    array<int, 26> cnt{};
    int ans = 0, n = s.size();
    for (int l = 0, r = 0; r < n; ++r) {
      ++cnt[s[r] - 'A'];
      while (r - l + 1 - *ranges::max_element(cnt) > k) {  // 不合法，從左邊縮
        --cnt[s[l] - 'A'];
        ++l;
      }
      ans = max(ans, r - l + 1);  // 縮完之後才記長度
    }
    return ans;
  }
};
```

> [!warning] 長度必須在**縮窗之後**才計算
> 把 `length` 先算好、縮完窗再拿舊值更新答案，是本題最常見的錯誤：
>
> ```cpp
> int length = r - l + 1;              // ✗ 縮窗前的長度
> while (窗不合法) { --cnt[s[l] - 'A']; ++l; }
> ans = max(ans, length);              // ✗ 記到的是「不合法的那個窗」
> ```
>
> 最小反例 `s = "BA"、k = 0`：`r=1` 時窗是 `"BA"`，`length` 已被算成 2；縮窗後窗只剩 `"A"`，但 `ans` 記的還是 2。正解是 1。
>
> 這個錯誤**只會讓答案偏大、不會偏小**，所以小測資常常矇混過關：`"AAB"、k=0` 得 3（正解 2）、`"BCBCA"、k=0` 得 2（正解 1）、`"ABBBCAB"、k=1` 得 5（正解 4）。

> [!tip] 不用擔心「窗會不會縮過頭」
> 長度 `≤ k+1` 的窗**必定合法**——至少有 1 個字元當多數，剩下最多 `k` 個用替換補掉。所以 `while` 的條件自己就會停，不需要額外加 `l < r - k` 之類的守衛（加了也不會錯，只是恆真的冗餘判斷）。同理答案下界天然就是 `min(n, k+1)`，不必特地預填初值。

### 方法二：窗永不縮小 — O(n)／O(1)

把問題換個問法：**維護一把只會變寬、絕不變窄的尺，掃完後它有多寬？**

之所以能永不變窄，是因為**縮窗對答案毫無幫助**：若長度 5 的窗不合法，縮到長度 3 才合法——但長度 3 的答案早在 `r` 更小時就記錄過了，縮窗只是在重新確認舊答案。

```cpp
// Time: O(n)
// Space: O(1)
class Solution {
 public:
  int characterReplacement(string s, int k) {
    array<int, 26> cnt{};
    int maxcnt = 0, l = 0, n = s.size();
    for (int r = 0; r < n; ++r) {
      maxcnt = max(maxcnt, ++cnt[s[r] - 'A']);  // 歷史最佳紀錄，只增不減
      if (r - l + 1 - maxcnt > k) {             // 不合法就整個窗右移，長度不變
        --cnt[s[l] - 'A'];
        ++l;
      }
    }
    return n - l;  // 窗長單調不減，最終窗長就是答案
  }
};
```

三個「為什麼」：

- **為什麼 `if` 不是 `while`**：`r` 每次只 +1，窗長最多超標 1，移一次 `l` 就回到原長。每步只有兩種結果——窗長 +1（試探成功）或整個窗右移一格（試探失敗）。
- **為什麼不用 `ans` 變數**：窗長單調不減，所以**最終窗長就是歷史最大窗長**；掃完時 `r = n-1`，窗是 `[l, n-1]`，長度 `n - l`。
- **為什麼窗卡住不縮也沒事**：卡住期間窗長沒有增加，那個「不合法的當前窗」從來沒被當成答案，它只是一把還在往右試的尺。

```txt
s = A A B A B B A ，k = 1

 r  窗口        窗長  maxcnt  動作
 0  [0,0] A       1     1     窗變長
 1  [0,1] AA      2     2     窗變長
 2  [0,2] AAB     3     2     窗變長
 3  [0,3] AABA    4     3     窗變長
 4  [0,4] AABAB   5     3     5−3=2 > 1，平移 l++（窗長仍是 5）
 5  [1,5] ABABB   5     3     平移
 6  [2,6] BABBA   5     3     平移
 => 回傳 n − l = 7 − 3 = 4
```

> [!important] `maxcnt` 只增不減為什麼不會出錯
> `maxcnt` 是「歷史最佳紀錄」而非「當前窗的真實最大值」，確實會過期。`s = "CCCAB"、k = 0` 走到 `r=4` 時窗是 `"CCAB"`，真實 max 只有 2，`maxcnt` 卻還停在 3。危險的只有一種情況：**`maxcnt` 偏大 → 判定過鬆 → 放行一個其實不合法的窗**。
>
> **先看放行到底需要什麼。** 直覺會以為需要 `maxcnt` 等於真實 max，其實只需要它**不高估**：
>
> ```txt
> 放行條件         窗長 − maxcnt ≤ k
> 若 maxcnt ≤ 真實max：
>                  窗長 − 真實max ≤ 窗長 − maxcnt ≤ k    → 窗真的合法 ✓
> ```
>
> 也就是說，`maxcnt` **只要是「某個字元在當前窗裡的真實計數」就夠了**，不必是最大的那個——它是真實 max 的下界，拿它判定只會更保守。
>
> **再看它憑什麼不高估。** 關鍵是「窗卡住時，判定用的窗長不會變」：
>
> ```txt
> r=4  窗[0,4] 長度5  5−3=2 > 1 判定失敗 → l++
> r=5  窗[1,5] 長度5  ← r 和 l 同時 +1，長度原地踏步
> r=6  窗[2,6] 長度5
> ```
>
> 上一步用長度 `L` 判定失敗，代表 `L − maxcnt > k`；這一步長度**還是 `L`**，要翻盤成 `L − maxcnt ≤ k`，**只剩一條路——`maxcnt` 必須變大**。而它變大只可能來自
>
> ```cpp
> maxcnt = max(maxcnt, ++cnt[s[r] - 'A']);
> //                   ^^^^^^^^^^^^^^^^^^ 破紀錄時 maxcnt 就等於這個值
> ```
>
> 正是 `s[r]` 在當前窗 `[l, r]` 裡貨真價實的計數。**卡住的窗要脫困，非得靠真數據刷新紀錄不可，過期值永遠沒機會放行。**

> [!note] 補完另一半：沒破紀錄也放行的情形
> 上面的論證只涵蓋「窗卡住過」。窗**從沒卡住過**時，`maxcnt` 不動也能一路成長（`k=2、s="ABC"`：窗從 1 長到 3，`maxcnt` 全程是 1）。但這種情形 `l` 根本沒移動過，各字元計數只增不減，當初設定的 `maxcnt` 現在只會**更小或相等**於真實計數——`maxcnt ≤ 真實max` 依然成立。
>
> 而「先卡住、之後卻沒破紀錄就放行」是不可能的：若中間平移過某一步 `u`，當時 `窗長_u − maxcnt > k`，窗長單調不減，故現在 `窗長 − maxcnt > k`，這步壓根不會放行。
>
> 窮舉驗證（字母集 3、長度 1~11、所有 `k`）：放行共 2604 萬次，其中破紀錄 1420 萬、沒破紀錄 1184 萬，而「沒破紀錄卻先前平移過」**0 次**；放行時 `maxcnt > 真實max` 也是 **0 次**。

> [!tip] 拿數字反駁一個常見誤解
> 「會不會有 `窗長=6`、真實 max`=2`、`maxcnt=4`、`k=3`？程式算 `6−4=2 ≤ 3` 放行，真值卻是 `6−2=4 > 3` 不合法。」
>
> `maxcnt=4` 要過期，必須曾在某個 `窗長_u ≤ 6` 判定失敗，即 `窗長_u − 4 > k`，於是 `6 − 4 > k` → **`k < 2`**。`k=3` 需要 `窗長_u > 7`，但窗長當時最多 6，**矛盾，不可達**。
>
> 窮舉確認：這個狀態確實存在，但**只出現在 `k=1`**（例如 `s="CCCCBBAA"` 走到窗 `[2,7]="CCBBAA"`），而 `k=1` 時 `6−4=2 > 1` 判定失敗、繼續平移，與真值判定一致。

> [!note] 該選哪個
> **方法一是首選。** 它跟 0003、0076 共用同一個骨架，換題只改一行判斷，而且正確性一眼可見。`n ≤ 10⁵` 下實測 1.5～2.6 ms，26 倍常數完全無感。
>
> 方法二快 13 倍，但它的正確性建立在上面那段不變量推理上——寫得出來不難，**面試時要當場說清楚為什麼對，才是難的**。它的真正價值是這個「窗永不縮小」框架可以原封不動搬到 [[1004-Max-Consecutive-Ones-III]] 這類「最長合法子陣列」題。

兩種寫法實測（50 萬組隨機測資對拍暴力解，字串長 1～26、字元集 1～26、`k` 涵蓋 `0..n`，加上 `""`、`"BA"`、`"AAB"`、`"CCCAB"`、`k = n` 等邊界，並在 ASan／UBSan 下重跑）全數通過。`n = 1e5` 效能：

```txt
                          字元集 2     字元集 26
  方法一（縮到合法）        2.56 ms      1.48 ms
  方法二（永不縮小）        0.19 ms      0.11 ms
```

## Related Problems

[[0003-Longest-Substring-Without-Repeating-Characters]] — 同一個骨架，收縮條件從「有重複」換成「窗長 − maxcnt > k」
[[1004-Max-Consecutive-Ones-III]] — 本題的二進位版（「最多翻 k 個 0」），方法二的框架可原封不動搬過去
[[1493-Longest-Subarray-of-1s-After-Deleting-One-Element]] — 上題再退化成 `k = 1` 且必須刪一個，邊界更刁
[[0076-Minimum-Window-Substring]] — 同骨架但求**最小**，改成窗口合法時才收縮，時機剛好相反
[[0567-Permutation-In-String]] — 窗長固定，退化成不需要 `while` 的定長滑動
