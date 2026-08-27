---
title: LeetCode Vault
dg-home: true
dg-publish: true
---

個人 LeetCode 解題筆記庫，用 Obsidian 管理、透過 Digital Garden 發佈。每題一篇筆記，收錄題目、C++ 解法（多種作法 + 複雜度）與關聯題目，並以間隔複習持續鞏固。

## 瀏覽方式

- **依模式分類**：筆記放在對應的演算法資料夾，資料夾以 NeetCode 150 roadmap 的學習順序加上數字前綴——`01-Arrays-and-Hashing`、`02-Two-Pointers`、`04-Binary-Search`、`06-Linked-List`、`09-Heap`、`12-1D-Dynamic-Programming`、`17-Bit-Manipulation` …，檔案總管即照學習路徑排序。
- **前後兩個附錄**：`00-Progress` 是間隔複習的進度儀表板（不發佈）；`99-Toolbox` 收跨題型的參考筆記——演算法模板、C++ 陷阱與量測方法，清單見下。

## Toolbox

`99-Toolbox` 是跨題型的模板與參考筆記，不綁單一題目（所以沒有 `leetcode-id`），由題目筆記以 wikilink 指過去。每篇都含原理圖解、完整實作、常見陷阱 callout 與選型表，**所有 C++ 都經過對拍驗證**，效能宣稱一律附實測數字。

**資料結構與區間查詢**

- [[Fenwick-Tree]] — 樹狀陣列，四種用法（單點加、區間加單點查、區間加區間和、樹上二分找第 k 小）
- [[Segment-Tree]] — 線段樹，遞迴／迭代／兩種 lazy
- [[Sparse-Table-RMQ]] — 靜態陣列的 O(1) 區間查詢，以及為什麼只能用於冪等運算
- [[Disjoint-Set-Union]] — 並查集與可撤銷版，兩個優化各自的效果實測
- [[Sqrt-Decomposition-and-Mos]] — 分塊與莫隊，處理線段樹做不到的不可合併查詢
- [[Monotonic-Stack]] — 兩側第一個更大／更小，四個變體、嚴格性陷阱與貢獻法

**圖論**

- [[Graph-Traversal-and-Connectivity]] — BFS／DFS／拓撲排序／環偵測／二分圖判定
- [[Shortest-Path-Algorithms]] — Dijkstra／0-1 BFS／Bellman-Ford／Floyd 的選型與陷阱
- [[Minimum-Spanning-Tree]] — Kruskal 與 Prim
- [[Strongly-Connected-Components]] — Tarjan／Kosaraju 與 2-SAT
- [[Max-Flow-and-Matching]] — Dinic、二分圖匹配、König 定理與建模技巧

**樹**

- [[Tree-Traversal-Iterative]] — 迭代版三序走訪與 Morris
- [[Binary-Lifting-LCA]] — 倍增祖先表求 LCA 與 k 級祖先
- [[Tree-Techniques]] — 直徑、重心、樹上差分、換根 DP

**字串・DP・數學・幾何**

- [[String-Matching]] — KMP、Z-function、字串雜湊與 anti-hash
- [[Knapsack-and-Classic-DP]] — 背包三變種與 LIS
- [[Binary-Search-Templates]] — 二分搜尋的邊界決策流程
- [[Modular-Arithmetic]] — 快速冪、模逆元、組合數與溢位界線
- [[Computational-Geometry-Basics]] — 叉積、線段相交、凸包，全整數運算

**C++ 與工程**

- [[STL-Pitfalls]] — 容器與迭代器的常見坑
- [[STL-List-Techniques]] — `std::list` 的 splice 等技巧
- [[Micro-Optimization-Myths]] — 微優化的實測檢驗
- [[Benchmarking-Toolkit]] — 怎麼量才算數（降雜訊、cachegrind、控制變因）
- [[Interview-English-Phrasebook]] — 面試英文用語

## 標籤

- **難度**：`easy`、`medium`、`hard`
- **題單**：`grind-169`、`neetcode-150`
- **模式**：`array`、`hash`、`two-pointers`、`monotonic-queue`、`backtracking`、`prefix-sum` …

## 筆記結構

每篇筆記的 frontmatter 含 `leetcode-id`、`difficulty`、`tags`、`memo`（一句話關鍵提示）與 `dg-publish`。內文固定三段：

- **Problem Description** — 原題敘述
- **Solution** — 一種以上解法，每段標注 `// Time:` / `// Space:`；關鍵洞見與陷阱以 callout（`> [!tip]`、`> [!warning]`）標示
- **Related Problems** — 以 `[[題號-標題]]` 連結相關題目

> 解題 / 複習進度**不**寫進筆記 frontmatter（以免每次複習都更動到發佈檔），改由 `/tutor` skill 記錄在 `00-Progress/` 目錄。該目錄有進版控，但不帶 `dg-publish`，所以只留在 repo 裡、不會發佈出去。

## 工具

- **新增題目**：`./newproblem --id <id> --title "<title>" --tags <tag> … [--dir <資料夾>]`，會依 `problemsets.toml`（Grind 169 / NeetCode 150）自動帶入難度並產生筆記骨架；`--dir` 可直接產在對應的模式資料夾下。
- **間隔複習**：`/tutor` skill——挑出到期題複習、盲解評分、MCQ 出題，並可從課程表挑新題。
