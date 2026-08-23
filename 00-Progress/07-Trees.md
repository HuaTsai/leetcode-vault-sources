# 07-Trees — SR Tracker

| Problem                                                                    | ID  | Difficulty | Box | Last Reviewed | Next Due | Reps | Lapses |
| -------------------------------------------------------------------------- | --- | ---------- | --- | ------------- | -------- | ---- | ------ |
| [[0098-Validate-Binary-Search-Tree\|Validate Binary Search Tree]]            | 98  | medium     | 0   | -             | -        | 0    | 0      |
| [[0104-Maximum-Depth-of-Binary-Tree\|Maximum Depth of Binary Tree]]          | 104 | easy       | 0   | -             | -        | 0    | 0      |
| [[0226-Invert-Binary-Tree\|Invert Binary Tree]]                              | 226 | easy       | 0   | -             | -        | 0    | 0      |
| [[0230-Kth-Smallest-Element-in-a-BST\|Kth Smallest Element in a BST]]        | 230 | medium     | 0   | -             | -        | 0    | 0      |

### Notes

#### 待理解（自陳，尚未測試）

**0230 Kth Smallest Element in a BST** — 2026-08-24 自陳：中序迭代解法能自己想到，但 **follow up（BST 頻繁 insert／delete 又要頻繁查 kth）想不到方向**。缺的那一步是「第 k 小是**排名查詢**，排名能從子樹大小算出來」，於是在節點上增廣 `size` 欄位，查詢從 O(k) 變成一路往下走的 O(h)；維護是免費的，因為只有根到該節點那條路徑會變，剛好是遞迴回程。另有一個誤解已澄清：**pbds 的 order statistic tree 不是「另一種資料結構」，它就是紅黑樹 + 這個 size 增廣**（policy 名稱 `tree_order_statistics_node_update`），實測手刻版與它同速；面試要講的是增廣本身而不是容器名字。還要記得補上另一半——增廣只解決「走 k 步」，`h` 要靠平衡樹才壓得到 O(log n)。下次測法：不給提示要求口述 follow up 的完整答案（增廣 + 平衡兩件事都要講到），或 blind-solve `kth` 與 `insert` 的 size 維護。

**0098 Validate Binary Search Tree** — 2026-08-24 自陳：上下界遞迴能自己寫出，但**初版用 `INT_MIN`／`INT_MAX` 當「無界」的哨兵**，在節點值剛好是極值時誤判（`[2147483647]` 回傳 false）。修正方向是讓「不存在」有自己的表示（`nullptr` 或 `optional<int>`），而不是去值域裡挑一個值。下次測法：出 output prediction，輸入單一節點 `[-2147483648]` 配哨兵版程式碼。

#### 其他

- 0104、0226 已有筆記但尚未測試（Box 0）。
- 三序迭代模板整理在 [[Tree-Traversal-Iterative]]，0098／0230 都建立在中序模板上，測這兩題前可先確認模板本身背熟了沒有。
