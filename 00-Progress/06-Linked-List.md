# 06-Linked-List — SR Tracker

| Problem                                                 | ID  | Difficulty | Box | Last Reviewed | Next Due   | Reps | Lapses |
| ------------------------------------------------------- | --- | ---------- | --- | ------------- | ---------- | ---- | ------ |
| [[0002-Add-Two-Numbers\|Add Two Numbers]]               | 2   | medium     | 0   | -             | -          | 0    | 0      |
| [[0021-Merge-Two-Sorted-Lists\|Merge Two Sorted Lists]] | 21  | easy       | 1   | 2026-08-24    | 2026-08-25 | 1    | 1      |
| [[0141-Linked-List-Cycle\|Linked List Cycle]]           | 141 | easy       | 0   | -             | -          | 0    | 0      |
| [[0143-Reorder-List\|Reorder List]]                     | 143 | medium     | 0   | -             | -          | 0    | 0      |
| [[0146-LRU-Cache\|LRU Cache]]                           | 146 | medium     | 0   | -             | -          | 0    | 0      |
| [[0206-Reverse-Linked-List\|Reverse Linked List]]       | 206 | easy       | 0   | -             | -          | 0    | 0      |

### Notes

**Merge Two Sorted Lists** — Confusion: 認為結尾用 `if / else if` 與三元運算子沒有差別，理由是「反正某一串已結束，把另一串接上即可」。這對非空輸入完全正確，但漏了 **list1 與 list2 一開始就都是 nullptr** 的情況。Key point: 兩串皆空時 `while` 不進入、兩個分支都不成立，`dummy.next` 從未被賦值，回傳值變成依賴 `ListNode` 預設建構子（LeetCode 剛好定義成 `next(nullptr)` 才安全，屬隱性外部依賴）。`cur->next = list1 ? list1 : list2;` 是無條件寫入，`nullptr` 本身就是空鏈的正解。下次改測「空輸入」這個邊界在其他 linked-list 題的表現，例如 0206 對空串列、或 0143 對單節點。
