# CLAUDE.md

個人 LeetCode 解題筆記庫，用 Obsidian 管理、透過 Digital Garden 發佈。每題一篇 Markdown 筆記。本檔給 AI agent 看，記錄改 / 加筆記時必須遵守的慣例。面向使用者的介紹見 `README.md`。

## 倉庫結構

- 筆記**依演算法模式分資料夾**，資料夾名是 `NN-Title-With-Dashes`，`NN` 是該模式在 **NeetCode 150 roadmap 上的位置**（2 位數補零、跳號編，現有：`01-Arrays-and-Hashing`、`02-Two-Pointers`、`05-Sliding-Window`、`09-Heap`、`10-Backtracking`、`12-1D-Dynamic-Programming`、`17-Bit-Manipulation`）。新增模式時用它在 roadmap 的號碼（完整 18 格見下），**不要把現有資料夾重編**。資料夾名用 dash、不用空格或 `&`，因為會發佈成 Digital Garden 的 URL 路徑。
  - roadmap 順序：01 Arrays & Hashing｜02 Two Pointers｜03 Stack｜04 Binary Search｜05 Sliding Window｜06 Linked List｜07 Trees｜08 Tries｜09 Heap / Priority Queue｜10 Backtracking｜11 Graphs｜12 1-D DP｜13 Intervals｜14 Greedy｜15 Advanced Graphs｜16 2-D DP｜17 Bit Manipulation｜18 Math & Geometry。
- 檔名格式：`NNNN-Title-With-Dashes.md`，`NNNN` 是 **4 位數補零的真實 LeetCode 題號**（由 `newproblem` 的 `sanitize_filename` 產生）。
- Obsidian `[[wikilink]]` 以 note 檔名（basename）解析、不看路徑，所以搬 / 改資料夾**不會弄壞連結**；`LeetCode.base` 用 `file.basename` 過濾、`newproblem` 一律寫在根目錄——三者都不受資料夾改名影響。
- `LeetCode.base`：Obsidian Base 互動式索引，依 `file.name` / `difficulty` / `tags` / `memo` 排序篩選。改欄位前先看這支。
- `problemsets.toml`：Grind 169 / NeetCode 150 的題名 → 難度對照表，`newproblem` 靠它自動帶難度與題單標籤。
- `Progress/` 與 `.obsidian/` 在 `.gitignore` 內。解題 / 複習進度**絕不**寫進筆記 frontmatter（避免每次複習都動到發佈檔），由 `/tutor` skill 記在 gitignored 的 `Progress/`。

## 筆記格式（改 / 加筆記時必守）

Frontmatter 欄位：`leetcode-id`、`difficulty`（easy/medium/hard）、`tags`（list）、`memo`、`dg-publish: true`。

內文固定三段：

1. **`## Problem Description`** — 原題英文敘述，**是使用者貼的原文，一律保持原樣**：不改寫、不潤飾、不翻譯、不增刪，兼修筆記時只動 `## Solution` 之後的內容。
2. **`## Solution`** — 先一段中文「核心觀念」帶出直覺，必要時加 ` ```txt ` 圖解；**一種以上解法並列**（`### 方法一`、`### 方法二`），每段標題附 `— O(?)／O(?)`，code block 開頭兩行 `// Time:` / `// Space:`。**第一個方法放最推薦 / 最正規的解**，使用者自己的解法若不同，放後面的方法但完整寫出。
3. **`## Related Problems`** — 用 Obsidian wikilink `[[NNNN-Title]]`（目標可尚未存在），每條後面接 `— 一句關聯理由`。

**Callout** 標關鍵洞見與陷阱：`> [!important]` / `> [!tip]` 放核心想法，`> [!warning]` 放陷阱，`> [!note]` 放補充。

**`memo`** = 一句繁體中文，點出核心技巧 + 一個關鍵洞見 / 易錯點。避免「基礎XX應用」這種空話。**YAML 安全**：只用全形標點（：；，／），絕不用 ASCII 的冒號加空格或 `#` 加空格（會破壞 plain scalar）。

## 工作流程（重要）

- 使用者通常會有兩種使用情境
  1. 貼一份 C++ 解法 + 問回饋 / 有沒有更好寫法，與使用者討論得到認可後，使用者同意後才改 `.md` 筆記
  2. 使用者已經寫好了 `.md` 答案，但想要幫忙檢查 code / memo / callout / 版面，這時候直接改 `.md` 檔案
- 修改文件請依照「筆記格式」規範，如果有不同的或者更優的作法，請在 `## Solution` 裡新增一個 H3 方法段落，並附上時間與空間複雜度。
- **任何要放進筆記的 C++ 都必須先在 `/tmp` 用 `g++ -std=c++20` 編譯並跑測資驗證**，正確性要驗證、不能用講的。筆記裡的 code 預設 `using namespace std;`、省略 include（LeetCode 風格）。
- 新增題目：`./newproblem --id <id> --title "<title>" --tags <tag>…`，會比對 `problemsets.toml` 自動帶難度 + 題單標籤並產生骨架。
- 改完 `.md` 後有 PostToolUse formatter 會自動重排（在 callout 補空行等），無害；但若要對同一區域再做 Edit，先重新 Read。
