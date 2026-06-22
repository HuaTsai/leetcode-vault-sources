---
leetcode-id: 36
difficulty: medium
tags:
  - array
  - hash
  - matrix
  - grind-169
  - neetcode-150
memo: row／column／box 各用一組集合偵測重複數字；訣竅是單趟掃完，box 編號用 (r/3)*3 + c/3 壓成 0~8
dg-publish: true
---

## Problem Description

Determine if a `9 x 9` Sudoku board is valid. Only the filled cells need to be validated according to the following rules:

1. Each row must contain the digits `1-9` without repetition.
2. Each column must contain the digits `1-9` without repetition.
3. Each of the nine `3 x 3` sub-boxes of the grid must contain the digits `1-9` without repetition.

Note:

- A Sudoku board (partially filled) could be valid but is not necessarily solvable.
- Only the filled cells need to be validated according to the mentioned rules.

## Solution

對 9 條 row、9 條 column、9 個 3×3 box 各自檢查「填入的數字 1-9 不重複」即可。因為盤面固定 9×9，時間／空間都是 O(1)。

> [!important] 關鍵：想辦法只走訪一次
> 最直覺的做法是 row / column / box 各掃一遍（方法一），整盤被走訪三次。但每個 cell 其實同時只屬於**一條 row、一條 column、一個 box**——所以一趟就能把三者一起紀錄、一起檢查（方法二）。把「多次掃描」收斂成「單趟掃描」是這題真正的觀察點，也是很多矩陣題共通的優化方向。

### 方法一：三函式分檢 — 直觀但掃三遍

把 row、column、box 各寫成一個檢查函式，每個用 `array<bool, 9>` 紀錄某數字是否出現過。缺點是整個盤面被掃了三遍、三個函式邏輯幾乎重複。

```cpp
// Time: O(1)
// Space: O(1)
class Solution {
 public:
  bool isvalidrow(const vector<vector<char>> &board, int r) {
    array<bool, 9> vals{};
    for (int c = 0; c < 9; ++c) {
      if (board[r][c] == '.') continue;
      if (vals[board[r][c] - '1']) return false;
      vals[board[r][c] - '1'] = true;
    }
    return true;
  }

  bool isvalidcolumn(const vector<vector<char>> &board, int c) {
    array<bool, 9> vals{};
    for (int r = 0; r < 9; ++r) {
      if (board[r][c] == '.') continue;
      if (vals[board[r][c] - '1']) return false;
      vals[board[r][c] - '1'] = true;
    }
    return true;
  }

  bool isvalidblock(const vector<vector<char>> &board, int r, int c) {
    array<bool, 9> vals{};
    for (int i = r; i < r + 3; ++i) {
      for (int j = c; j < c + 3; ++j) {
        if (board[i][j] == '.') continue;
        if (vals[board[i][j] - '1']) return false;
        vals[board[i][j] - '1'] = true;
      }
    }
    return true;
  }

  bool isValidSudoku(vector<vector<char>>& board) {
    for (int r = 0; r < 9; ++r) {
      if (!isvalidrow(board, r)) return false;
    }
    for (int c = 0; c < 9; ++c) {
      if (!isvalidcolumn(board, c)) return false;
    }
    for (int r = 0; r < 9; r += 3) {
      for (int c = 0; c < 9; c += 3) {
        if (!isvalidblock(board, r, c)) return false;
      }
    }
    return true;
  }
};
```

> [!warning] `std::array` 的 `{}` 不能省
> `array<bool, 9> vals{}` 那對 `{}` 是正確性關鍵。`std::array` 是 aggregate、沒有 constructor，寫成 `array<bool, 9> vals;`（區域變數）時每個 `bool` 都是**不確定值**，讀取屬於 UB，重複判斷會時對時錯。加上 `{}` 觸發 value-initialization，才會全部歸零成 `false`。
> 對比 `std::vector`：`vector<bool> v(9)` 是靠「帶 size 的 constructor」幫你 value-initialize，所以不必補 `{}`；但 `vector<bool> v;` 是空的（size 0），並非 9 個 `false`。
> 另外儲存期會影響結果：全域／static 的 array 會先被零初始化，只有**區域變數**才會踩到這個陷阱。

### 方法二：單趟掃完 — 27 組紀錄一次更新

換個角度：整盤其實就是 **27 個群組**（9 row + 9 column + 9 box），每個群組都要求 1-9 不重複。那就一趟掃過 81 格，對每個 cell 同時更新它所屬的 row / column / box 三個群組。用一張 `27 × 9` 的表 `hasvalue` 統一存：`0~8` 是 row、`9~17` 是 column、`18~26` 是 box。

> [!tip] 三組索引怎麼算
>
> - row：`rid = i`
> - column：`cid = 9 + j`
> - box：`bid = 18 + i / 3 * 3 + j / 3`——`i/3`、`j/3` 是 box 的列號／行號（0~2），`*3` 把列號攤平成 0~8，再加上 `18` 的偏移落到 box 區段。例如 `(i,j)=(4,7)` → `18 + 1*3 + 2 = 23`。

```cpp
// Time: O(1)
// Space: O(1)
class Solution {
 public:
  bool isValidSudoku(vector<vector<char>>& board) {
    vector<vector<bool>> hasvalue(27, vector<bool>(9, false));
    for (int i = 0; i < 9; ++i) {
      for (int j = 0; j < 9; ++j) {
        if (board[i][j] == '.') continue;
        int v = board[i][j] - '1';
        int rid = i, cid = 9 + j, bid = 18 + i / 3 * 3 + j / 3;
        if (hasvalue[rid][v] || hasvalue[cid][v] || hasvalue[bid][v])
          return false;
        hasvalue[rid][v] = hasvalue[cid][v] = hasvalue[bid][v] = true;
      }
    }
    return true;
  }
};
```

把三條規則統一成「27 個群組都不能有重複」，概念乾淨、整盤只掃一遍。唯一可斟酌的是 `0 / 9+ / 18+` 這組偏移稍微 magic；若想更自我說明，也可改成三個具名陣列 `rows / cols / boxes`（各 `array<bool, 9>`）、用 `b = (i/3)*3 + j/3` 當 box 索引——兩者等價，看你偏好緊湊還是直白。

## Related Problems

[[0037-Sudoku-Solver]] — 直接延伸：用回溯真正去解數獨，過程中要反覆做同樣的合法性檢查
[[0217-Contains-Duplicate]] — 同樣是「用集合偵測某群組內有無重複」的核心，本題對 27 個群組各做一次
[[0073-Set-Matrix-Zeroes]] — 另一個經典的矩陣遍歷 + 標記題
