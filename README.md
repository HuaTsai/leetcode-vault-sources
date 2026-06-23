---
title: LeetCode Vault
dg-home: true
dg-publish: true
---

個人 LeetCode 解題筆記庫，用 Obsidian 管理、透過 Digital Garden 發佈。每題一篇筆記，收錄題目、C++ 解法（多種作法 + 複雜度）與關聯題目，並以間隔複習持續鞏固。

## 瀏覽方式

- **依模式分類**：筆記放在對應的演算法資料夾，資料夾以 NeetCode 150 roadmap 的學習順序加上數字前綴——`01-Arrays-and-Hashing`、`02-Two-Pointers`、`05-Sliding-Window`、`09-Heap`、`10-Backtracking`、`12-1D-Dynamic-Programming`、`17-Bit-Manipulation` …，檔案總管即照學習路徑排序。
- **互動式索引**：用 Obsidian 開啟 `LeetCode.base`，可依題號 / 難度 / 標籤 / memo 排序、篩選整份題庫。

## 標籤

- **難度**：`easy`、`medium`、`hard`
- **題單**：`grind-169`、`neetcode-150`
- **模式**：`array`、`hash`、`two-pointers`、`monotonic-queue`、`backtracking`、`prefix-sum` …

## 筆記結構

每篇筆記的 frontmatter 含 `leetcode-id`、`difficulty`、`tags`、`memo`（一句話關鍵提示）與 `dg-publish`。內文固定三段：

- **Problem Description** — 原題敘述
- **Solution** — 一種以上解法，每段標注 `// Time:` / `// Space:`；關鍵洞見與陷阱以 callout（`> [!tip]`、`> [!warning]`）標示
- **Related Problems** — 以 `[[題號-標題]]` 連結相關題目

> 解題 / 複習進度**不**寫進筆記 frontmatter（以免每次複習都更動到發佈檔），改由 `/tutor` skill 記錄在 gitignored 的 `Progress/` 目錄。

## 工具

- **新增題目**：`./newproblem --id <id> --title "<title>" --tags <tag> …`，會依 `problemsets.toml`（Grind 169 / NeetCode 150）自動帶入難度並產生筆記骨架。
- **間隔複習**：`/tutor` skill——挑出到期題複習、盲解評分、MCQ 出題，並可從課程表挑新題。
