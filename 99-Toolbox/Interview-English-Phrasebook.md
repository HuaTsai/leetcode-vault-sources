---
tags:
  - interview
  - english
  - reference
dg-publish: true
---

## 這篇在解決什麼

這不是英文教材。這裡只收**「讀得懂、但當場講不出來」**的那些——你看到 `l = m + 1` 完全知道它在幹嘛，但要用嘴巴講給面試官聽的那三秒鐘，腦袋是空的。

收錄標準有兩條：

1. **唸不出來的**：符號、發音陷阱、把一整段 code 講成人話。
2. **知道這個字就省一整句的**：不知道 `in-place`，你得說「不用額外的陣列，直接改原本那個」；知道了就是兩個字。

> [!important] 面試不是唸符號，是講意圖
> 最常見的失分不是文法錯，是把 code **逐字唸出來**：「el equals em plus one」——面試官聽完還是不知道你在想什麼。正確的是講那一行的**目的**：「I move `left` past `mid`, since everything up to `mid` is too small.」
>
> 這條比這篇裡任何一個單字都重要。**簡單但正確的句子，永遠贏過複雜但講錯的句子。**

## 一、符號怎麼唸

| 符號 | 唸法 | 備註 |
| --- | --- | --- |
| `a[i]` | "a of i" ／ "a sub i" | "a bracket i" 也通但較少 |
| `a[i][j]` | "a of i, j" | 別唸成 "a bracket i bracket j" |
| `->` | "arrow" | `p->next` = "p arrow next" |
| `::` | "colon colon" ／ "scope resolution" | |
| `==` | "equals" ／ "double equals" | |
| `=` | "set X to Y" ／ "assign" | 講賦值別說 "equals"，會跟 `==` 混 |
| `!=` | "not equal to" | |
| `!x` | "not x" | |
| `&&` `\|\|` | "and" ／ "or" | |
| `&` `\|` | "bitwise and" ／ "bitwise or" | 一定要加 bitwise，不然聽成邏輯運算 |
| `^` | "XOR"（唸 "ex-or"） | 別唸 "caret"，那是符號名不是運算名 |
| `~` | "tilde"（TIL-də）／ "bitwise not" | |
| `<<` `>>` | "left shift" ／ "right shift" | `x << 1` = "x shifted left by one" |
| `%` | "mod" ／ "modulo" | `i % n` = "i mod n" |
| `n / 2` | "n over two" | 整數除法要補一句 "integer division" |
| `i++` | "increment i" | 比 "i plus plus" 專業 |
| `+=` | "plus equals" ／ "add ... to ..." | |
| `*p` | "dereference p" ／ "star p" | |
| `&x` | "address of x" | |
| `n²` `2ⁿ` `√n` | "n squared" ／ "two to the n" ／ "square root of n" | |
| `O(n log n)` | "big oh of n log n" | log 唸 /lɒɡ/，不要拼成 L-O-G |
| `()` `[]` `{}` `<>` | parentheses ／ (square) brackets ／ curly braces ／ angle brackets | 口語常說 "parens" |
| `_` | "underscore" | |
| `i` `j` `k` `n` `m` `q` | "eye, jay, kay, en, em, cue" | |
| `l` `r` | **直接說 "left" ／ "right"** | `l` 唸 "el" 跟 `1` 難分辨，別冒險 |

## 二、真的會唸錯的字

| 字 | 唸法 | 常見錯誤 |
| --- | --- | --- |
| queue | 「kyoo」 | 後面 `ueue` 不發音 |
| deque | 「deck」 | 不是 "dee-queue" |
| cache | 「cash」 | 不是 "ca-shay" |
| vertex / vertices | 「VER-teks」／「VER-ti-seez」 | 複數不是 "vertexes" |
| index / indices | 「IN-deks」／「IN-di-seez」 | indexes 也可接受 |
| pseudocode | 「SOO-do-code」 | `p` 不發音 |
| contiguous | 「con-TIG-you-us」 | 講「記憶體連續」用它，不是 continuous |
| adjacent | 「a-JAY-sent」 | |
| bipartite | 「bye-PAR-tite」 | |
| asymptotic | 「a-sim-TOT-ic」 | |
| amortized | 「AM-or-tized」 | |
| heuristic | 「hyoo-RIS-tik」 | `h` 要發音 |
| recursion | 「ri-KUR-zhun」 | |
| height | 「hite」 | 跟 weight 不同韻 |

## 三、知道這個字，就省一整句

這節是本篇的重點：左邊是你想表達的東西，右邊是那個字，最右邊是**不知道它的時候你會怎麼卡**。

| 想說 | 英文 | 沒這個字你會變成 |
| --- | --- | --- |
| 不用額外空間、直接改原陣列 | **in-place** | "without creating a new array, we just change the original one..." |
| 平均下來是 O(1) | **amortized** O(1) | "most of the time it's fast, but sometimes it's slow, and if you average it..." |
| 樹退化成一條鏈 | the tree **degenerates** into a chain | "when the tree becomes very very deep and thin like a list..." |
| 迴圈每一輪都成立的性質 | **loop invariant** | "this thing is always true every time we come back to the top..." |
| 反例 | **counterexample** | "an example that shows it's wrong" |
| 邊界情況 | **edge case** | "special situations like empty or only one element" |
| 剪枝 | **prune** the branch | "we don't need to keep searching this part" |
| 提早結束 | **bail out early** ／ **short-circuit** | "we can stop immediately without checking the rest" |
| 記憶化 | **memoize** ／ memoization | "we save the answer so we don't compute it again" |
| 哨兵 / 虛擬頭節點 | **dummy node** ／ sentinel | "a fake node in front so we don't need a special case for the head" |
| 用空間換時間 | **trade space for time** | "we use more memory to make it faster" |
| 已經不可能更快了 | this is **optimal** | "there's no way to make it faster, because you have to look at every element anyway" |
| 去重 | **deduplicate** ／ remove duplicates | |
| 攤平巢狀結構 | **flatten** | |
| 前綴和 | **prefix sum** | |
| 快慢指標 | **fast and slow pointers**（別名 tortoise and hare） | |
| 單調堆疊 | **monotonic stack** | |
| 遞推式 | **recurrence relation** | "the formula that gets the next one from the previous ones" |
| 自底向上 / 自頂向下 | **bottom-up (tabulation)** ／ **top-down (memoization)** | |
| 越界 | go **out of bounds** ／ read **past the end** | |
| 溢位 | **overflow** | |

### 樹

| 中文 | 英文 | 備註 |
| --- | --- | --- |
| 葉節點 | leaf（複數 **leaves**） | |
| 深度 / 高度 | depth（往下數）／ height（往上數） | 面試官會追問差別，先想好 |
| 祖先 / 後代 | ancestor ／ descendant | |
| 最近共同祖先 | **LCA** = lowest common ancestor | |
| 不跨父節點的路徑 | **downward path** ／ 正式講法 **ancestor–descendant path** | 見 [[0124-Binary-Tree-Maximum-Path-Sum]] |
| 在某節點折返的路徑 | the path **bends** at `u` ／ `u` is the **highest node** of the path | |
| 前 / 中 / 後序 | preorder ／ inorder ／ postorder traversal | |
| 層序 | level-order traversal | |
| 退化的樹 | **skewed** ／ degenerate tree | |

> [!tip] full / complete / perfect 三個字很常被混用，先講清楚再用
> **full**（每個節點有 0 或 2 個孩子）、**complete**（除最後一層外都滿，最後一層靠左）、**perfect**（每一層都滿）。不確定對方的定義時，直接描述比用術語安全：「a tree where every level is completely filled」。

### 圖

| 中文 | 英文 | 備註 |
| --- | --- | --- |
| 環 | **cycle** | **不要說 loop**，那是迴圈 |
| 鄰居 | neighbor ／ adjacent node | |
| 入度 / 出度 | in-degree ／ out-degree | |
| 連通分量 | connected component | |
| 拓撲排序 | topological sort ／ "topo sort" | |
| 鄰接表 / 矩陣 | adjacency list ／ adjacency matrix | |

## 四、把一整段 code 講成人話

反例與正解對照。同一段二分搜尋：

```cpp
int l = 0, r = nums.size() - 1;
while (l <= r) {
  int m = l + (r - l) / 2;
  if (nums[m] == target) return m;
  if (nums[m] < target) l = m + 1;
  else r = m - 1;
}
return -1;
```

**✗ 逐字唸（面試官聽不出你懂不懂）**

> "int el equals zero, int ar equals nums dot size minus one. While el less than or equal ar, int em equals el plus r minus l over two. If nums of em equals target, return em..."

**✓ 講意圖（同樣長度，資訊量完全不同）**

> "I keep a **closed interval** `[left, right]` that always contains the answer if it exists. Each round I look at the midpoint — I write it as `left + (right - left) / 2` **to avoid overflow** rather than `(left + right) / 2`. If it's the target I'm done. If it's too small, the answer must be **strictly to the right**, so I move `left` past `mid`; otherwise I move `right` below `mid`. If the interval becomes empty, the target isn't there, so I return minus one."

差別在三處，每一處都是面試官在打分的地方：

1. 說出**不變量**（"always contains the answer"），不只說步驟。
2. 說出**為什麼這樣寫**（"to avoid overflow"），這是主動送分。
3. 說出**為什麼可以丟掉一半**（"must be strictly to the right"），證明你不是背模板。

模板本身見 [[Binary-Search-Templates]]。

## 五、面試流程句型

### 開場釐清（問了不扣分，不問才扣）

- "Let me make sure I understand the problem — you want ..., right?"
- "**How large can n get?**" ← 這句直接告訴你該做 O(n log n) 還是 O(n)
- "Can I assume the input is non-empty / sorted / fits in a 32-bit integer?"
- "Are there duplicates?" ／ "Can the values be negative?"
- "What should I return for an empty input?"
- "Is it okay if I modify the input in place?"

### 提出思路（先講再寫，永遠）

- "Let me walk you through my approach before I write any code."
- "The **brute-force** would be to check every pair — that's O(n²). Let me see if I can do better."
- "The **key observation** is that ..."
- "I'll **trade space for time** here — a hash map makes the lookup O(1)."
- "Let me start with something correct and optimize afterwards."
- "This reduces to a problem I've seen before — it's basically ..."

### 邊寫邊講

- "I'll pull this out into a helper so the main function stays readable."
- "I'm handling the empty case up front so the loop below doesn't need to worry about it."
- "I'll call this `seen` — it holds the values we've already visited."
- "This is the part I mentioned earlier where ..."

### 講複雜度（主動講，別等人問）

- "This runs in **O(n log n) time**, **dominated by the sort**."
- "Space is O(h), where h is the height — O(log n) if the tree is balanced, O(n) in the worst case when it degenerates into a chain."
- "The **amortized** cost per operation is O(1), even though a single resize is O(n)."
- "O(n) is optimal here — you have to look at every element at least once."

### 手動跑一次

- "Let me **trace through** an example to make sure this is right."
- "Say the input is `[1, 2, 3]`. First iteration, `i` is zero, so ..."
- "Let me check the edge cases: empty input, a single element, all negatives."

### 卡住的時候（重點：不要安靜）

- "Let me **take a step back**." ← 萬用，爭取時間又聽起來有控制感
- "I'm not seeing the optimal solution yet — let me **think out loud** for a second."
- "I have an O(n²) approach. I suspect there's a linear one using a hash map — **does that sound like the right direction?**"
- "**Am I on the right track?**" ／ "Could you give me a nudge?"
- 「我不知道」不要講成 "I don't know"，講成：**"I'm not sure yet, but here's what I do know: ..."**

### 發現自己寫錯

- "Actually, hold on — that breaks if the array has duplicates."
- "**Good catch.** Let me fix that."
- "I need to move this line above the loop."
- 不要道歉超過一次。"Let me fix that" 就夠了，`sorry` 說三次只會顯得慌。

### 討論取捨與收尾

- "There's a **trade-off**: the hash map gives O(1) lookup but costs O(n) extra space. If memory were tight I'd use the two-pointer version."
- "It's faster in practice, but **asymptotically** the same."
- "I think that's it. Want me to go through the complexity again, or try another test case?"

> [!warning] 卡住時最糟的反應是安靜
> 面試官看不到你腦袋裡在跑什麼——沉默 30 秒對他來說跟「不會」沒有區別。**即使還沒有想法，也要出聲描述你正在排除什麼**：「I don't think two pointers works here because the array isn't sorted, so I'm thinking about what I could precompute instead.」這句話零解法含量，但它證明了你在推理。

## 六、中式英文陷阱

| 想說 | ✗ | ✓ |
| --- | --- | --- |
| 存進 map | "save into the map" | "**store** it in the map" ／ "put it in the map" |
| 用 map 記錄次數 | "record the times" | "use a map to **keep track of** the counts" |
| 這個陣列沒排序 | "the array is disordered / unordered" | "the array is **unsorted**"（unordered 專指 hash 的無序） |
| 時間複雜度是 O(n) | "the complexity is O(n) times" | "**it runs in O(n) time**" |
| 跑迴圈 | "run a cycle" | "run a **loop**" ／ "iterate"（cycle 在圖論是環） |
| 這一題 | "this topic" | "this **problem**" ／ "this question" |
| 指標會越界 | "the pointer will out of range" | "the index will **go out of bounds**" |
| 暴力解 | — | "the **brute-force** approach" |
| 假設輸入是排好的 | "suppose that if the input sorted" | "**assuming** the input is sorted" ／ "let's say ..." |
| 這樣會變慢 | "the efficiency is low" | "that would **push it to** O(n²)" ／ "that's too slow" |
| n 的範圍 | "the range of n" | "the **constraints** on n"（range 也通，constraints 更貼題目用語） |
| 差不多一樣 | "so so" | "**roughly** the same" |
| 我覺得可以 | "I think can" | "I think **that works**" |
| 順便一提 | "by the way" 沒錯，但面試裡更自然 | "**one thing worth noting** — ..." |

> [!note] 講錯了不用回頭修文法
> 面試官在評估的是解題能力，不是托福口說。時態錯、冠詞漏掉、單複數不一致，**沒有人在扣分**。真正會被扣分的只有兩種：讓對方**聽不懂你的演算法**，以及**用錯術語導致意思相反**（例如把 cycle 說成 loop、把 unsorted 說成 unordered）。這篇的六節裡，只有第三節和第六節值得背，其他查得到就好。

## Related Problems

- [[Binary-Search-Templates]] — 第四節那段示範 code 的完整模板與推導
- [[Tree-Traversal-Iterative]] — preorder／inorder／postorder 這幾個字對應的實作
- [[0124-Binary-Tree-Maximum-Path-Sum]] — downward path／LCA 這組術語的出處
- [[STL-Pitfalls]] — 面試被追問「為什麼不用這個容器」時的彈藥
