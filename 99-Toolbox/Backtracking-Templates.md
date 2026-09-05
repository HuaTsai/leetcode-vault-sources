---
tags:
  - backtracking
  - pattern
dg-publish: true
---

## 這篇在解決什麼

回溯的骨架短到十行，而且看起來只有一種：**選一個 → 遞迴 → 撤銷**。但真正寫的時候會發現有兩套長得不一樣的骨架（`for` 迴圈枚舉候選 vs 每個元素選／不選），以及一堆「該傳 `i` 還是 `i + 1`」「答案在哪一層收」「重複值怎麼跳」的分岔。

這些分岔**不是要背的，它們由題目的三個屬性決定**。這篇先給決策流程，再逐一收模板與各自的陷阱。

全部對拍驗過：子集／組合／排列／選不選／遞增子序列各 2000 組隨機測資對照 2ⁿ 暴力枚舉與 `next_permutation`；切割型 2000 組對照 2ⁿ⁻¹ 切點枚舉；網格回溯 3000 組對照「枚舉所有格子序列」；N-Queens 對照已知解數 `n = 1..9 → 1,0,0,2,10,4,40,92,352`，全過。

## 決策流程

```txt
問題一：答案是「所有節點」還是「只有葉節點」？
  ├─ 所有節點（子集型）        → 進門就收，不需要 base case
  └─ 只有葉節點（組合／排列型）→ 到達終止條件才收

問題二：往下傳什麼？（決定了這是哪一類問題）
  ├─ bt(i)          → 同一個元素可以重複取         組合，不計順序
  ├─ bt(i + 1)      → 每個下標只用一次              組合，不計順序
  └─ 全枚舉 + used[] → 每個下標只用一次、但換序算新解  排列，計順序

問題三：候選有重複值嗎？
  ├─ 沒有 → 不必去重
  └─ 有   → 排序後同層跳過（排列另配 used，不可排序改本層 hash set）
```

## 通用骨架

```txt
void bt(狀態) {
  if (該收答案) ans.push_back(cur);          // 子集型在這裡，其餘配 return
  for (每個候選 c) {
    if (c 不合法 或 c 是同層重複) continue;   // 剪枝與去重都擋在遞迴之前
    做選擇(c);            // cur.push_back / used[i] = true / 就地標記
    bt(推進後的狀態);
    撤銷選擇(c);          // 必須和「做選擇」嚴格配對
  }
}
```

> [!important] 剪枝與去重一定要擋在遞迴之前
> 寫成「先遞迴進去、在子呼叫開頭才判斷不合法然後 return」是能跑的，但那棵子樹的入口成本已經付掉了。實測 `candidates = {2,3,5,7,11,13,17,19,23}`、`target = 40`（291 組解）：靠 `remain < 0` 在子呼叫回頭要 **11803** 次遞迴，把 `nums[i] > remain` 提到迴圈裡擋掉只要 **2649** 次。

## 模板一：子集型 — 每個節點都是答案

```cpp
// Time: O(n2^n)  2^n 個子集，每組複製 O(n)
// Space: O(n)    遞迴堆疊 + cur，不計輸出
class Solution {
 public:
  vector<vector<int>> subsets(vector<int>& n) { nums = n; bt(0); return ans; }

 private:
  void bt(int start) {
    ans.push_back(cur);  // 進門就收，沒有 base case
    for (int i = start; i < (int)nums.size(); ++i) {
      cur.push_back(nums[i]);
      bt(i + 1);
      cur.pop_back();
    }
  }

  vector<int> nums, cur;
  vector<vector<int>> ans;
};
```

`start` 單調遞增就是「不計順序」的全部祕密：`[1,2]` 只會從 `1 → 2` 這條路長出來，`2 → 1` 那條被 `start` 擋死了。**這也順帶說明了排列型為什麼不能有 `start`** —— 排列要的正是被擋掉的那些。

## 模板二：組合型 — 只有葉節點是答案

### 2a 可重複取：傳 `i`

```cpp
// Time: 由 target 與最小候選決定，上界 O(n^(target/min))
// Space: O(target/min)  遞迴深度
class Solution {
 public:
  vector<vector<int>> combinationSum(vector<int>& c, int target) {
    nums = c;
    ranges::sort(nums);  // 只為了讓剪枝能用 break
    bt(0, target);
    return ans;
  }

 private:
  void bt(int start, int remain) {
    if (remain == 0) { ans.push_back(cur); return; }
    for (int i = start; i < (int)nums.size(); ++i) {
      if (nums[i] > remain) break;  // 排序後，後面只會更大
      cur.push_back(nums[i]);
      bt(i, remain - nums[i]);      // i 而非 i + 1：同一個還能再取
      cur.pop_back();
    }
  }

  vector<int> nums, cur;
  vector<vector<int>> ans;
};
```

> [!tip] 排序把剪枝從 `continue` 升級成 `break`
> 兩者砍掉的**遞迴子樹完全一樣**，差別在還要不要空轉迴圈。同上組測資：`continue` 版遞迴 2649 次但迴圈跑滿 **11802** 次，`break` 版遞迴同樣 2649 次、迴圈只剩 **4908** 次。遞迴次數騙不了人的地方在於它看不到迴圈裡的空轉。

### 2b 每個下標用一次 + 重複值去重：傳 `i + 1`

```cpp
// Time: O(n2^n)  最壞退化成每個元素選／不選
// Space: O(n)
void bt(int start, int remain) {
  if (remain == 0) { ans.push_back(cur); return; }
  for (int i = start; i < (int)nums.size(); ++i) {
    if (nums[i] > remain) break;
    if (i > start && nums[i] == nums[i - 1]) continue;  // 同層只讓第一個代表
    cur.push_back(nums[i]);
    bt(i + 1, remain - nums[i]);
    cur.pop_back();
  }
}
```

> [!warning] 是 `i > start`，不是 `i > 0`
> 這是整個回溯家族最高頻的一個 bug：
>
> | 條件 | `nums[i - 1]` 的身分 | 該不該跳 |
> | --- | --- | --- |
> | `i > start` | 本層剛遞迴完又 `pop_back` 撤銷的兄弟，子樹已整棵走過 | 跳 |
> | `i == start` | 上一層選走的，**還留在 `cur` 裡** | 不能跳，這正是「同值取兩個」 |
>
> 寫成 `i > 0` 會把後者一起砍掉，所有「同一個值取兩個以上」的解全部消失。實測見 [[0090-Subsets-II]] 與 [[0040-Combination-Sum-II]]。

## 模板三：排列型 — 沒有 `start`，改用 `used[]`

```cpp
// Time: O(n·n!)  n! 個排列，每組複製 O(n)
// Space: O(n)    used + cur + 遞迴深度
class Solution {
 public:
  vector<vector<int>> permuteUnique(vector<int>& n) {
    nums = n;
    ranges::sort(nums);  // 有重複值才需要；讓相同值相鄰
    used.assign(nums.size(), false);
    bt();
    return ans;
  }

 private:
  void bt() {
    if (cur.size() == nums.size()) { ans.push_back(cur); return; }
    for (int i = 0; i < (int)nums.size(); ++i) {   // 每層都從 0 開始
      if (used[i]) continue;
      if (i > 0 && nums[i] == nums[i - 1] && !used[i - 1]) continue;  // 同層去重
      used[i] = true;
      cur.push_back(nums[i]);
      bt();
      cur.pop_back();
      used[i] = false;  // 兩個撤銷都不能漏
    }
  }

  vector<int> nums, cur;
  vector<bool> used;
  vector<vector<int>> ans;
};
```

> [!important] 排列的「同層」不能用 `i > start` 表達
> 排列每層都從 0 掃到 n-1，沒有 `start` 可以當基準。這時判斷「`nums[i-1]` 是同層兄弟還是祖先」的依據換成 `used[i - 1]`：
>
> - `!used[i - 1]` → 前一個相同值**不在當前路徑上**，代表它要嘛還沒輪到、要嘛剛被撤銷 → 同層兄弟 → 跳
> - `used[i - 1]` → 前一個相同值**還在 `cur` 裡**，是祖先 → 不跳，這是同值取兩個
>
> 語意跟 `i > start` / `i == start` 完全對應，只是換了偵測手段。

> [!note] `used[i-1]` 版也正確，但慢 4 倍
> 把條件寫成 `nums[i] == nums[i-1] && used[i-1]` 也能得到正確答案 —— 它保留的是「一串相同值裡的最後一個當代表」而不是第一個。但它必須先讓前面的重複值全部進來才知道要跳，等於在樹的深處才剪。實測 `nums = [1,1,1,2,2,3,3,3,4]`（5040 組解）：`!used[i-1]` 版遞迴 **15227** 次，`used[i-1]` 版 **66298** 次。**兩個都對，但要記就記 `!used[i-1]`。**

## 模板四：選／不選 — 二元遞迴

不枚舉「下一個拿誰」，改成對每個下標獨立回答「要不要」。樹是二元的、深度固定 n，答案一律在葉節點。

```cpp
// Time: O(n2^n)
// Space: O(n)
void bt(int i) {
  if (i == (int)nums.size()) { ans.push_back(cur); return; }
  cur.push_back(nums[i]);
  bt(i + 1);        // 選
  cur.pop_back();
  bt(i + 1);        // 不選
}
```

有重複值時，去重規則跟模板一**長得完全不一樣**：

```cpp
void bt(int i) {
  if (i == (int)nums.size()) { ans.push_back(cur); return; }
  cur.push_back(nums[i]);
  bt(i + 1);
  cur.pop_back();

  int j = i;  // 不選 nums[i] → 連同所有等值的一起跳過
  while (j < (int)nums.size() && nums[j] == nums[i]) ++j;
  bt(j);
}
```

理由：一串 k 個相同值，唯一有意義的決策是「取幾個」。若允許「跳過第一個卻選第二個」，它跟「選第一個跳過第二個」會產出同一個集合。

> [!warning] 模板一和模板四不能混用
> 兩套骨架**各自都已經走遍整個解空間**。把 `for (i = start; ...)` 套在「兩次 `bt(i+1)`」外面，等於同一個子集被數很多次 —— 而且**跟有沒有重複值無關**。實測 `nums = [1,2,3,4,5]`（五個相異值）：正確 32 組，混用版吐出 **162** 組。
>
> 判斷法很簡單：**一個 `bt` 裡出現兩次遞迴呼叫，就不該再有 `for`；有 `for`，迴圈裡就只該有一次遞迴呼叫。**

## 重複值去重總表

| 骨架 | 去重條件 | 為什麼 |
| --- | --- | --- |
| 模板一／二（有 `start`） | 排序 + `i > start && nums[i] == nums[i-1]` → `continue` | 同層兄弟的子樹是前一個的真子集 |
| 模板三（排列，有 `used`） | 排序 + `i > 0 && nums[i] == nums[i-1] && !used[i-1]` → `continue` | 沒有 `start`，改用 `used[i-1]` 分辨兄弟／祖先 |
| 模板四（選／不選） | 不選時 `while (nums[j] == nums[i]) ++j` 跳過整段 | 一串相同值只該決策「取幾個」 |
| **不能排序**（要保留原順序，如遞增子序列） | 本層開一個 `unordered_set<int> seen` | 排序會破壞題意，只能在同層記錄「這個值本層試過了」 |

最後一種的樣子（`seen` 是**區域變數**，每層一個，絕不能提成成員）：

```cpp
// Time: O(n2^n)
// Space: O(n)  每層一個 seen，深度 O(n)
void bt(int start) {
  if (cur.size() >= 2) ans.push_back(cur);
  unordered_set<int> seen;  // 只管本層
  for (int i = start; i < (int)nums.size(); ++i) {
    if (!cur.empty() && nums[i] < cur.back()) continue;  // 題目的合法性條件
    if (seen.count(nums[i])) continue;                   // 同層去重
    seen.insert(nums[i]);
    cur.push_back(nums[i]);
    bt(i + 1);
    cur.pop_back();
  }
}
```

> [!warning] `seen` 提成成員變數會錯得很難查
> 它必須隨著層一起生一起滅。做成成員（或傳參考共用）等於跨層去重，會把合法的「深處再取一次同值」也砍掉，而且小測資通常看不出來。同理，`seen` **不需要在回溯時移除元素** —— 離開這層它就該整個消失。

## 換個東西枚舉：三個常見變體

骨架不變，變的只是「候選是什麼」。

### 切割型 — 枚舉切在哪裡

```cpp
// Time: O(n2^n)  2^(n-1) 種切法，每種驗回文 O(n)
// Space: O(n)
void bt(int start) {
  if (start == (int)s.size()) { ans.push_back(cur); return; }
  for (int end = start; end < (int)s.size(); ++end) {  // 候選 = 這一段切到哪
    if (!pal(start, end)) continue;
    cur.push_back(s.substr(start, end - start + 1));
    bt(end + 1);
    cur.pop_back();
  }
}
```

`end + 1` 對應模板二的 `i + 1`：下一段從這段的結尾之後開始，切點單調遞增，天然不重複。

### 網格型 — 就地標記代替 `visited`

```cpp
// Time: O(mn·3^L)  L 為字串長度，每步最多 3 個新方向
// Space: O(L)      遞迴深度，不另開 visited
bool bt(int r, int c, int k) {
  if (r < 0 || r >= m || c < 0 || c >= n || b[r][c] != w[k]) return false;
  if (k + 1 == (int)w.size()) return true;
  char save = b[r][c];
  b[r][c] = 0;  // 做選擇：標記為已在路徑上
  bool ok = bt(r + 1, c, k + 1) || bt(r - 1, c, k + 1) ||
            bt(r, c + 1, k + 1) || bt(r, c - 1, k + 1);
  b[r][c] = save;  // 撤銷
  return ok;
}
```

> [!tip] 邊界檢查放在被呼叫端，不是呼叫端
> 四個方向各寫一次 `if (r+1 < m && ...)` 會膨脹成一團。統一在函式開頭一次擋掉越界與不匹配，四個遞迴呼叫就能寫成一行 `||`。這也是回溯裡少數該「先進去再判斷」的例外 —— 它省下的是四份重複的邊界條件。

### 棋盤型 — 用衝突集合取代掃描

```cpp
// Time: O(n!)  上界；實際被三個衝突集合剪掉大半
// Space: O(n)
void bt(int r) {
  if (r == n) { ++cnt; return; }
  for (int c = 0; c < n; ++c) {
    if (col[c] || d1[r + c] || d2[r - c + n]) continue;  // O(1) 判衝突
    col[c] = d1[r + c] = d2[r - c + n] = true;
    bt(r + 1);
    col[c] = d1[r + c] = d2[r - c + n] = false;          // 三個都要撤銷
  }
}
```

一列放一個皇后，所以「同列」自動排除；`r + c` 是主對角線編號、`r - c + n` 是副對角線（`+n` 避免負數索引）。把「往回掃已放的皇后」O(n) 換成三個布林陣列 O(1)。

## 複雜度速查

| 模板 | 節點數 | 每個答案的成本 | 總計 |
| --- | --- | --- | --- |
| 子集型 | 2ⁿ | O(n) 複製 | O(n·2ⁿ) |
| 組合型（用一次） | ≤ 2ⁿ | O(n) | O(n·2ⁿ) |
| 組合型（可重複取） | 由 target 決定 | O(target/min) | O(n^(target/min)) 上界 |
| 排列型 | n! | O(n) | O(n·n!) |
| 切割型 | 2ⁿ⁻¹ | O(n) 驗證 + 複製 | O(n·2ⁿ) |

> [!note] 回溯的複雜度是「節點數 × 每節點成本」，不是「答案數 × 成本」
> 剪枝掉的節點也走過入口。可重複取的那類尤其寫不出緊界 —— 面試時講清楚「上界是這樣，實際被剪枝壓到多少取決於資料」比硬套一個公式誠實。

## 題目分類

| 模板 | 題目 |
| --- | --- |
| 一 · 子集型 | [[0078-Subsets]]、[[0090-Subsets-II]] |
| 二 · 組合型 | [[0039-Combination-Sum]]、[[0040-Combination-Sum-II]]、[[0077-Combinations]]、[[0216-Combination-Sum-III]] |
| 三 · 排列型 | [[0046-Permutations]]、[[0047-Permutations-II]] |
| 四 · 選／不選 | [[0078-Subsets]]、[[0090-Subsets-II]] |
| 切割型 | [[0131-Palindrome-Partitioning]]、[[0093-Restore-IP-Addresses]] |
| 網格型 | [[0079-Word-Search]]、[[0212-Word-Search-II]] |
| 棋盤型 | [[0051-N-Queens]]、[[0037-Sudoku-Solver]] |
| 不可排序去重 | [[0491-Non-decreasing-Subsequences]] |
| 候選來自映射 | [[0017-Letter-Combinations-of-a-Phone-Number]] |

## Related Problems

[[0090-Subsets-II]] — 同時是模板一與模板四的代表，也是「兩套骨架混用」反例的來源
[[0040-Combination-Sum-II]] — `i > start` 去重規則講得最完整的一題，附不去重改用 set 過濾的成本實測
[[0039-Combination-Sum]] — 可重複取（傳 `i`）的原型，也是回溯與完全背包 DP 的分界點
[[0078-Subsets]] — 最乾淨的子集骨架，無重複值、不必去重
[[Knapsack-and-Classic-DP]] — 當題目從「列舉所有組合」改問「有幾種／最優值」，回溯就該換成 DP
[[Graph-Traversal-and-Connectivity]] — DFS 骨架同源，差別在圖走訪的 `visited` 不撤銷、回溯的一定要撤銷
