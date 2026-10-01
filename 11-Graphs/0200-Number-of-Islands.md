---
leetcode-id: 200
difficulty: medium
tags:
  - array
  - depth-first-search
  - breadth-first-search
  - union-find
  - grind-169
  - neetcode-150
memo: 掃到陸地就計數並把整座島沉成 0 當 visited；DSU 版只需 unite 上與左兩個方向，components 從陸地數起算而非 m·n
dg-publish: true
---

## Problem Description

Given an `m x n` 2D binary grid `grid` which represents a map of `'1'`s (land) and `'0'`s (water), return the number of islands.

An island is surrounded by water and is formed by connecting adjacent lands horizontally or vertically. You may assume all four edges of the grid are all surrounded by water.

## Solution

這題就是「數連通塊」：把每格陸地當節點、上下左右相鄰的陸地之間有邊，答案是連通塊個數。做法只有一個骨架——**掃到一格還沒看過的陸地就 `++count`，然後把跟它連通的整座島標記掉**，之後再掃到這座島的其他格子就不會重複計數。「標記掉」用 DFS、BFS 或 DSU 都行，差別在空間與寫法。

```txt
1 1 0 0 0        掃到 (0,0)：count=1，把這座島沉掉 →   0 0 0 0 0
1 1 0 0 0                                              0 0 0 0 0
0 0 1 0 0        掃到 (2,2)：count=2，沉掉 →           0 0 0 0 0
0 0 0 1 1        掃到 (3,3)：count=3，沉掉 →           0 0 0 0 0
```

> [!important] 不用額外 visited 陣列：直接把走過的 `'1'` 改成 `'0'`
> 「沉島」就是 visited 標記，而且沉掉的格子之後再被外層迴圈掃到時自然跳過。如果題目不准改輸入，再開一個 `vector<vector<bool>>`。

### 方法一：DFS 沉島 — O(mn)／O(mn)

對每一格陸地遞迴往四個方向擴散，把整座島沉成 `'0'`。邊界檢查全部放在遞迴入口，呼叫端就不用自己判斷。

```cpp
// Time: O(mn)，每格最多被沉一次、被檢查常數次
// Space: O(mn)，遞迴深度最差是整張圖都是陸地時的蛇形路徑
class Solution {
 public:
  int numIslands(vector<vector<char>>& grid) {
    int m = grid.size(), n = grid[0].size(), count = 0;
    for (int i = 0; i < m; ++i) {
      for (int j = 0; j < n; ++j) {
        if (grid[i][j] == '1') {
          ++count;
          sink(grid, i, j);
        }
      }
    }
    return count;
  }

 private:
  void sink(vector<vector<char>>& grid, int i, int j) {
    if (i < 0 || i >= ssize(grid) || j < 0 || j >= ssize(grid[0]) || grid[i][j] != '1') {
      return;
    }
    grid[i][j] = '0';
    sink(grid, i + 1, j);
    sink(grid, i - 1, j);
    sink(grid, i, j + 1);
    sink(grid, i, j - 1);
  }
};
```

> [!note] 遞迴深度
> 題目上限 300×300，全陸地時遞迴深度最多 9×10⁴ 層，每層只有三個參數，實測在預設 8 MB stack 下沒問題。但如果格子數到 10⁶ 級，就該換方法二的 BFS 或用顯式 stack。

### 方法二：BFS — O(mn)／O(min(m, n))

同一個骨架，把遞迴換成 queue。**入隊時就沉掉**，不是出隊時才沉，否則同一格會被四個鄰居各推一次。

```cpp
// Time: O(mn)
// Space: O(min(m, n))，queue 最多同時裝一條「波前」，寬度不超過短邊
class Solution {
 public:
  int numIslands(vector<vector<char>>& grid) {
    int m = grid.size(), n = grid[0].size(), count = 0;
    constexpr int dirs[5] = {0, 1, 0, -1, 0};  // 相鄰兩項就是一組 (dr, dc)
    queue<pair<int, int>> q;
    for (int i = 0; i < m; ++i) {
      for (int j = 0; j < n; ++j) {
        if (grid[i][j] != '1') {
          continue;
        }
        ++count;
        grid[i][j] = '0';
        q.push({i, j});
        while (!q.empty()) {
          auto [r, c] = q.front();
          q.pop();
          for (int d = 0; d < 4; ++d) {
            int nr = r + dirs[d], nc = c + dirs[d + 1];
            if (nr >= 0 && nr < m && nc >= 0 && nc < n && grid[nr][nc] == '1') {
              grid[nr][nc] = '0';  // 入隊即標記
              q.push({nr, nc});
            }
          }
        }
      }
    }
    return count;
  }
};
```

> [!warning] 出隊才標記會重複入隊
> 若改成 `pop` 之後才 `grid[r][c] = '0'`，一格陸地在被處理前可能已經被上下左右四個鄰居各 push 一次，queue 會膨脹到 4 倍、答案仍對但常數變差；在「計數」類題目這不會錯，但在最短路類題目就會多算距離。

### 方法三：DSU 並查集 — O(mn·α(mn))／O(mn)

把格子 `(i, j)` 編號成 `i * n + j`，每格陸地先讓 `components` 加一，再跟**上方與左方**的陸地 `unite`——只看兩個方向就夠，因為掃描順序保證每條邊剛好被走到一次。這題用 DSU 是殺雞用牛刀，它真正的價值在**動態加陸地**的 [[0305-Number-of-Islands-II]]：每放一格就能 O(α) 回答目前有幾座島，DFS／BFS 做不到。

```cpp
// Time: O(mn·α(mn))，α 是反 Ackermann 函數，實務上視為常數
// Space: O(mn)，parent 與 sz 各一份
struct DSU {
  vector<int> parent, sz;
  int components = 0;

  explicit DSU(int n) : parent(n), sz(n, 1) {
    iota(parent.begin(), parent.end(), 0);
  }

  int find(int x) {
    return parent[x] == x ? x : parent[x] = find(parent[x]);
  }

  bool unite(int a, int b) {
    a = find(a), b = find(b);
    if (a == b) {
      return false;
    }
    if (sz[a] < sz[b]) {
      swap(a, b);
    }
    parent[b] = a;
    sz[a] += sz[b];
    --components;
    return true;
  }
};

class Solution {
 public:
  int numIslands(vector<vector<char>>& grid) {
    int m = grid.size(), n = grid[0].size();
    DSU dsu(m * n);
    for (int i = 0; i < m; ++i) {
      for (int j = 0; j < n; ++j) {
        if (grid[i][j] != '1') continue;
        ++dsu.components;  // 只有陸地算一個集合
        if (i > 0 && grid[i - 1][j] == '1') dsu.unite(i * n + j, (i - 1) * n + j);
        if (j > 0 && grid[i][j - 1] == '1') dsu.unite(i * n + j, i * n + j - 1);
      }
    }
    return dsu.components;
  }
};
```

> [!tip] `components` 從 0 開始、見陸地才加，不是從 `m * n` 開始
> DSU 開了 `m * n` 個節點，但水格子從頭到尾不參與 `unite`，若用 Toolbox 模板預設的 `components = n` 起算，最後會多出所有水格子的數量。要嘛像這裡從 0 累加，要嘛先數陸地格數 `e`、建一張 `(i, j) → 0..e-1` 的對照表再開 `DSU(e)`——後者多一趟掃描和一個 `int` 矩陣，換到的只是陸地稀疏時省一點 `parent` 空間。

> [!note] 只 unite 上與左
> 外層是 row-major 掃描，掃到 `(i, j)` 時 `(i-1, j)` 與 `(i, j-1)` 一定已經處理過，而 `(i+1, j)`、`(i, j+1)` 之後輪到它們時會回頭連上 `(i, j)`。四個方向都寫只是每條邊多 `unite` 一次並回傳 `false`，結果不變。

三種寫法已互相對拍：題目範例、20000 組 1–8×1–8 隨機 grid（陸地密度 0–100%）、300×300 全陸地，結果全部一致。

## Related Problems

- [[0695-Max-Area-of-Island]] — 同一個沉島骨架，`sink` 改成回傳面積再取最大
- [[0130-Surrounded-Regions]] — 反過來從邊界的 `'O'` 出發沉島，剩下的才是被包圍的
- [[1254-Number-of-Closed-Islands]] — 先沉掉碰到邊界的島，再數剩下的
- [[0417-Pacific-Atlantic-Water-Flow]] — 從兩組邊界各做一次多源 DFS／BFS，取交集
- [[0305-Number-of-Islands-II]] — 動態加陸地，DSU 才是正解的那一題
- [[0547-Number-of-Provinces]] — 鄰接矩陣版的連通塊計數，DFS 與 DSU 都適用
- [[Disjoint-Set-Union]] — 路徑壓縮與按大小合併為什麼缺一不可
- [[Graph-Traversal-and-Connectivity]] — 連通塊計數的通用骨架
