---
leetcode-id: 79
difficulty: medium
tags:
  - array
  - string
  - backtracking
  - depth-first-search
  - grind-169
  - neetcode-150
memo: 網格回溯 DFS，原地把走過的格子覆寫成井字號、回溯時用 `word[k]` 還原就不必另開 visited；標記一定要撤銷，因為同一格可能被另一條路徑用到；先比字頻並從較稀有的一端開始搜，題目上限的最壞測資從 29 ms 降到 1 µs 以下
dg-publish: true
---

## Problem Description

Given an `m x n` grid of characters `board` and a string `word`, return `true` if `word` exists in the grid.

The word can be constructed from letters of sequentially adjacent cells, where adjacent cells are horizontally or vertically neighboring. The same letter cell may not be used more than once.

## Solution

核心觀念：從每一格當起點做 DFS，第 `k` 步只問一件事——這一格是不是 `word[k]`。是就把它標成「已在路徑上」、往四個方向找 `word[k + 1]`；四個方向都失敗就**撤銷標記**再回頭。這是 [[Backtracking-Templates]] 的網格型：候選是「下一步走哪一格」，做選擇是標記、撤銷是還原。

```txt
board        word = "ABCCED"

A B C E      路徑：(0,0)A → (0,1)B → (0,2)C → (1,2)C → (2,2)E → (2,1)D
S F C S
A D E E

走完 k = 2、準備找 word[3] = 'C' 時的 board（'#' = 已在路徑上）

# # # E      (0,2) 的四個鄰居：上 越界、左 '#'、右 'E'、下 'C' ← 只有這格符合
S F C S
A D E E
```

> [!important] 標記一定要撤銷，這是回溯和圖走訪的分界
> [[0200-Number-of-Islands]] 那類走訪的 `visited` 只進不出，因為它問的是「這格到不到得了」，到過一次就有答案。這題問的是「有沒有一條**不重複用格子的路徑**」，同一格在這條路徑上失敗，換一條路徑來可能就成功，所以離開時必須把格子還回去。漏掉撤銷不會當掉，只會讓某些存在的答案回傳 `false`。

以下三個方法的複雜度共用符號：**`m`、`n`** = board 的列數與行數、**`L`** = `word.size()`。起點有 `m·n` 個，第一步 4 個方向，之後每一步扣掉來的方向最多 3 個，所以是 `O(m·n·3^L)`。

### 方法一：原地標記回溯 — O(m·n·3^L)／O(L)

```cpp
// Time: O(m·n·3^L)  m·n 個起點，每步最多 3 個新方向
// Space: O(L)       遞迴深度，不另開 visited
class Solution {
 public:
  bool exist(vector<vector<char>>& board, string word) {
    for (int i = 0; i < ssize(board); ++i) {
      for (int j = 0; j < ssize(board[0]); ++j) {
        if (dfs(board, word, i, j, 0)) return true;
      }
    }
    return false;
  }

 private:
  bool dfs(vector<vector<char>>& board, const string& word, int i, int j,
           int k) {
    if (i < 0 || i >= ssize(board) || j < 0 || j >= ssize(board[0]) ||
        board[i][j] != word[k]) {
      return false;  // 越界、字元不符、已在路徑上（'#'）一次擋掉
    }
    if (k + 1 == ssize(word)) return true;
    board[i][j] = '#';  // 做選擇：標記為已在路徑上
    bool found =
        dfs(board, word, i + 1, j, k + 1) || dfs(board, word, i - 1, j, k + 1) ||
        dfs(board, word, i, j + 1, k + 1) || dfs(board, word, i, j - 1, k + 1);
    board[i][j] = word[k];  // 撤銷：能走到這裡代表原本就是 word[k]
    return found;
  }
};
```

> [!tip] `'#'` 同時扮演 visited，所以一個條件擋三件事
> 題目保證 board 和 word 只有英文字母，`'#'` 永遠不會等於任何 `word[k]`，於是「已在路徑上」自動被 `board[i][j] != word[k]` 擋掉，不需要另外判斷。還原時也不用先把原字元存起來——能通過開頭那道檢查，代表這格原本就是 `word[k]`。

> [!warning] 找到答案也要還原
> `found` 為 `true` 時直接 `return true` 看起來省一步，但 board 是以參考傳進來的，這樣會把一路的 `'#'` 留在呼叫者的資料裡。先還原再回傳，函式結束後 board 才會跟進來時一模一樣（驗證時每組測資都比對過）。

### 方法二：字頻剪枝 + 從稀有端開始 — O(m·n·3^L)／O(L)

`dfs` 和方法一完全相同，只在 `exist` 開頭多做兩件事。最壞複雜度沒變，但它砍的是整棵搜尋樹的**入口**，實測差距是數量級的。

```cpp
// Time: O(m·n·3^L)  最壞不變；字頻不足時 O(m·n + L) 直接結束
// Space: O(L)       遞迴深度；cnt 是固定 256 格
class Solution {
 public:
  bool exist(vector<vector<char>>& board, string word) {
    array<int, 256> cnt{};
    for (const auto& row : board) {
      for (unsigned char c : row) ++cnt[c];
    }
    auto remain = cnt;
    for (unsigned char c : word) {
      if (--remain[c] < 0) return false;  // 供給不夠，連搜都不用搜
    }
    if (cnt[(unsigned char)word.front()] > cnt[(unsigned char)word.back()]) {
      ranges::reverse(word);  // 從較稀有的一端出發，起點候選比較少
    }
    for (int i = 0; i < ssize(board); ++i) {
      for (int j = 0; j < ssize(board[0]); ++j) {
        if (dfs(board, word, i, j, 0)) return true;
      }
    }
    return false;
  }

 private:
  bool dfs(vector<vector<char>>& board, const string& word, int i, int j,
           int k) {
    if (i < 0 || i >= ssize(board) || j < 0 || j >= ssize(board[0]) ||
        board[i][j] != word[k]) {
      return false;
    }
    if (k + 1 == ssize(word)) return true;
    board[i][j] = '#';
    bool found =
        dfs(board, word, i + 1, j, k + 1) || dfs(board, word, i - 1, j, k + 1) ||
        dfs(board, word, i, j + 1, k + 1) || dfs(board, word, i, j - 1, k + 1);
    board[i][j] = word[k];
    return found;
  }
};
```

> [!important] 為什麼可以把 word 反過來搜
> 相鄰是對稱的：`a` 在 `b` 隔壁，`b` 就在 `a` 隔壁。所以一條拼出 `word` 的路徑倒著走，就是一條拼出反轉字串的路徑，兩者存在與否完全等價。回溯最怕的是「前面一大段都配得上、最後一個字才失敗」，把稀有字元換到開頭，失敗就會發生在樹根而不是樹葉。

> [!note] 字頻檢查涵蓋了「word 比 board 長」
> `word.size() > m * n` 時，word 的字元總需求必然超過 board 的總供給，至少有一個字元會讓 `remain` 變負，所以不必另外寫長度檢查。

> [!note] 實測：題目上限的最壞測資
> 量測條件：`g++ -std=c++20 -O2`，board 6×6、word 長度 15（皆為題目上限），各跑 11 次取中位數；另以 20 萬組隨機測資對「用 `set` 記座標的暴力列舉」交叉驗證，三個方法輸出全數一致。
>
> **測資 A**：board 全是 `A`，word = `A`×14 + `B`（board 裡沒有 `B`）
>
> | 版本               | `dfs` 呼叫次數 | 時間       |
> | ------------------ | -------------- | ---------- |
> | 方法一 原地標記    | 9,030,868      | 29.3 ms    |
> | 方法二 字頻 + 反轉 | **0**          | **< 1 µs** |
> | 方法三 visited     | 4,120,732      | 32.1 ms    |
>
> **測資 B**：board 全是 `A`、對角兩個角落各一顆 `B`，word = `A`×13 + `BB`（字頻檢查過得了，但兩顆 `B` 不相鄰）
>
> | 版本               | `dfs` 呼叫次數 | 時間       |
> | ------------------ | -------------- | ---------- |
> | 方法一 原地標記    | 3,983,932      | 13.1 ms    |
> | 方法二 字頻 + 反轉 | **44**         | **< 1 µs** |
> | 方法三 visited     | 1,879,120      | 14.4 ms    |
>
> 測資 A 靠字頻檢查、測資 B 靠反轉，兩個剪枝各自負責一種情形。測資 B 的 44 次 = 36 個起點各一次，加上兩顆 `B` 各往四個方向試一次。
>
> 另一個觀察：方法一的呼叫次數是方法三的兩倍多（它連越界和已走過的格子都先呼叫進去再擋），時間卻沒有比較慢。被擋掉的呼叫只做幾個比較就返回，成本遠低於一次真正的展開——**呼叫次數少不等於比較快**，[[0040-Combination-Sum-II]] 量到過同樣的現象。

### 方法三：visited 陣列，不修改輸入 — O(m·n·3^L)／O(m·n)

我原本的寫法。多花 `O(m·n)` 的空間，換到的是 **board 全程唯讀**：`dfs` 收的是 `const` 參考。面試被追問「board 不能改、或是多個執行緒共用同一份 board 怎麼辦」，答案就是這個版本。

```cpp
// Time: O(m·n·3^L)
// Space: O(m·n)  visited；遞迴深度 O(L) 且 L <= m·n
class Solution {
 public:
  bool exist(vector<vector<char>>& board, string word) {
    m = board.size();
    n = board[0].size();
    if (ssize(word) > m * n) return false;
    visited.assign(m, vector<bool>(n));
    for (int i = 0; i < m; ++i) {
      for (int j = 0; j < n; ++j) {
        if (dfs(board, word, i, j, 0)) return true;
      }
    }
    return false;
  }

 private:
  bool dfs(const vector<vector<char>>& board, const string& word, int i, int j,
           int k) {
    if (board[i][j] != word[k]) return false;
    if (k + 1 == ssize(word)) return true;
    static constexpr array<int, 5> dir{0, 1, 0, -1, 0};
    visited[i][j] = true;
    bool found = false;
    for (int x = 0; x < 4 && !found; ++x) {
      int ii = i + dir[x];
      int jj = j + dir[x + 1];
      found = ii >= 0 && ii < m && jj >= 0 && jj < n && !visited[ii][jj] &&
              dfs(board, word, ii, jj, k + 1);
    }
    visited[i][j] = false;
    return found;
  }

  vector<vector<bool>> visited;
  int m, n;
};
```

> [!tip] `dir{0, 1, 0, -1, 0}` 一個陣列編四個方向
> 取相鄰兩格 `(dir[x], dir[x + 1])` 依序得到 `(0,1)`、`(1,0)`、`(0,-1)`、`(-1,0)`，也就是右、下、左、上。比開兩個 `dx`／`dy` 陣列少記一份。

> [!warning] `dir` 不能寫成函式裡的區域 `vector`
> 我最初寫的是 `vector<int> dir{0, 1, 0, -1, 0};`，放在 `dfs` 裡面。`vector` 的內容在 heap 上，等於**每一次遞迴都配置再釋放一次**。同樣的測資 A／B，呼叫次數完全相同，時間卻差了將近一倍：
>
> | `dir` 的寫法                     | 測資 A      | 測資 B      |
> | -------------------------------- | ----------- | ----------- |
> | 區域 `vector<int>`               | 56.2 ms     | 25.5 ms     |
> | `static constexpr array<int, 5>` | **32.1 ms** | **14.4 ms** |
>
> 這是整份程式裡唯一量得到的常數成本。呼應 [[Micro-Optimization-Myths]] 的結論：該關心的是配置與記憶體存取，不是那幾個 cycle 的算術。

> [!note] 原地標記並沒有比 visited 快
> 方法一和方法三的時間差在 10% 左右（29.3 vs 32.1 ms、13.1 vs 14.4 ms），另一輪量測甚至出現過反過來的結果，不足以當選型理由。選方法一是因為省空間、程式碼短；選方法三是因為不能動輸入。

## Related Problems

- [[0212-Word-Search-II]] — 一次找多個單字，把所有單字建成 Trie 後用同一套原地標記回溯，一趟 DFS 同時比對
- [[0200-Number-of-Islands]] — 同樣是網格四方向 DFS，但標記只進不出；對照著看就是「走訪」與「回溯」的差別
- [[0130-Surrounded-Regions]] — 一樣原地改寫格子當標記，走訪完再依標記還原成最終答案
- [[0046-Permutations]] — `used[]` 的標記與撤銷和這題的 visited 同構，只是候選從「四個鄰居」換成「所有還沒用的元素」
- [[Backtracking-Templates]] — 本題屬「網格型」，邊界檢查放在被呼叫端的理由見該篇
- [[Graph-Traversal-and-Connectivity]] — DFS 骨架同源，差別在圖走訪的 `visited` 不撤銷、回溯的一定要撤銷
- [[Micro-Optimization-Myths]] — 方法三量到的 heap 配置成本，是該篇「真正貴的是配置」的又一個實例
