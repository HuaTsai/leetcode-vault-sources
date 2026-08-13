---
tags:
  - cpp
  - stl
  - linked-list
  - pattern
dg-publish: true
---

## 這篇在解決什麼

`std::list` 在競賽與刷題裡幾乎沒人用——大多數時候它確實是錯的選擇。但它有一個**其他標準容器都給不了的保證**，一旦題目需要那個保證，它就從「不該用」變成「只能用它」。[[0146-LRU-Cache]] 就是那個場景。

這篇分兩半：先講清楚為什麼平常不該用（附實測數字），再講那個唯一的理由，以及跟著它來的一整套只有 `list` 才有的成員函式。

## 一、先講成本：三個實測

### 走訪慢 7 倍

```txt
純走訪 1000 萬個 int，-O2
  vector   2.99 ms
  list    21.77 ms      慢 7.3 倍
```

`vector` 的元素連續排列，硬體預取器可以一路猜對；`list` 的節點四散在 heap 上，每跳一步都可能是 cache miss，而且**下一個位址要等這一步讀回來才知道**，連預取都無從預取起。

### 每個節點多背兩根指標

```txt
                       payload   每節點    額外
  list<int>                 4      24      20      ← 4 bytes 資料，20 bytes 開銷
  list<pair<int,int>>       8      24      16
  forward_list<int>         4      16      12
  vector<pair<int,int>>     8       8       0      ← 對照組
```

`list<int>` 一個節點 24 bytes 裝 4 bytes 資料，**六分之五是開銷**（兩根 8 bytes 指標 + 對齊）。這也是走訪慢的第二個原因：同樣的 cache line 裝得下的有效資料少 3 倍。

### 「中間插入 list 比較快」這句話有前提

這是最常被誤傳的一句。實測 10 萬次在中間插入：

```txt
  list：手上已經有 iterator          1.00 ms
  list：每次都要 advance 走到中間     18633.93 ms      ← 慢 18000 倍
  vector：memmove 搬一半的資料         369.85 ms
```

> [!important] `list` 的 O(1) 插入只算「已經站在那個位置」之後的動作
> 走到那個位置本身是 O(n)，而且是**沒有任何硬體加速的 O(n)**——一次一個 cache miss。`vector` 的 O(n) 插入是 `memmove`，連續搬移，是 CPU 最擅長的事。所以「中間插入」這件事上 `vector` 反而快 50 倍。
>
> **只有當那個 iterator 是別的結構（通常是 hash map）直接遞給你的時候，`list` 才贏。**這正是 LRU cache 的形狀。

## 二、唯一無可取代的理由：位址穩定

`list` 真正的賣點不是插入快，是**元素的位址與 iterator 永遠不會因為別處的增刪而失效**：

```txt
list：大量 push_front / push_back / erase 之後
  *it 還是 20，&*it 沒變
splice 搬到另一個 list 之後
  *it 還是 20，&*it 沒變，而且 it 現在屬於新的容器

對照 vector：只是 reserve 一下
  指標就變了（整塊重新配置）
```

拿標準容器對照：

| 容器            | 中間 O(1) 增刪 | 元素位址穩定       | iterator 穩定    |
| --------------- | -------------- | ------------------ | ---------------- |
| `vector`        | ✗              | ✗ 擴容就全變       | ✗                |
| `deque`         | ✗              | 只有兩端增刪時穩定 | ✗ 任何增刪都失效 |
| `map` / `set`   | ✗ 位置由序決定 | ✓                  | ✓                |
| `unordered_map` | ✗ 位置由雜湊定 | ✓                  | ✗ rehash 就失效  |
| **`list`**      | **✓**          | **✓**              | **✓**            |

**只有 `list` 同時具備「位置由你決定」與「位址永不變動」。**需要「把某個元素從序列中間拔起來換位置，同時外部還握著指向它的把手」時，沒有第二個選擇。

> [!tip] 判準一句話
> 你是不是**從別的容器拿到 iterator，然後直接對它動手**？是就用 `list`，其餘情況先假設 `vector` 是對的。

## 三、`splice`：把節點接到別處，不搬資料

`splice` 是 `list` 的招牌，也是上面那個保證的具體用法。它把節點從一條串列摘下來接到另一條，**只重接指標**，不配置、不釋放、不複製也不移動元素本身。標準明文保證被搬動的 iterator／指標／參考全部繼續有效，只是所屬容器換了。

三種多載：

```cpp
lst.splice(pos, other);                 // (a) 整條 other 接到 pos 之前，other 變空
lst.splice(pos, other, it);             // (b) 只搬 other 的一個節點
lst.splice(pos, other, first, last);    // (c) 搬 [first, last) 這一段
```

`other` 可以就是 `lst` 自己，用來在同一條串列裡搬動——LRU 的 `data_.splice(data_.end(), data_, it)` 就是這種用法。

實測（搬 100 萬個節點，`-O2`）：

| 多載                           | 標準要求的複雜度       | libstdc++ 實測 |
| ------------------------------ | ---------------------- | -------------- |
| (a) `splice(pos, other)`       | O(1)                   | 0.0003 ms      |
| (b) `splice(pos, other, it)`   | O(1)                   | 4.8 ns／次     |
| (c) `splice(pos, *this, f, l)` | **O(1)**（同一條串列） | **2.26 ms**    |
| (d) `splice(pos, other, f, l)` | O(distance(f, l))      | 3.30 ms        |

> [!warning] 區間版即使搬的是自己，libstdc++ 也是線性
> (c) 標準規定 `addressof(x) == this` 時是常數時間，實測卻跟 (d) 同一個量級。原因在 `stl_list.h` 的實作**無條件**先數一遍：
>
> ```cpp
> size_t __n = _S_distance(__first, __last);   // ← 不管是不是同一條串列都算
> this->_M_inc_size(__n);
> __x._M_dec_size(__n);
> this->_M_transfer(...);                      // 這行才是真正的重接，O(1)
> ```
>
> 這是 **C++11 起 `size()` 必須是 O(1)** 的代價：`list` 要自己記著元素數，區間 splice 就得知道搬了幾個。把 size 追蹤關掉（`-D_GLIBCXX_USE_CXX11_ABI=0`）驗證，(c) 立刻從 2.26 ms 掉到 0.0004 ms，證實線性成本全來自這筆記帳。
>
> 實務結論：**單節點版 (b) 才是可以放心當 O(1) 用的那個**，也就是 LRU 需要的那個。要搬一整段就假設它是 O(n)。

```txt
splice(lst.end(), lst, it) 做的事：把 it 那個節點的前後接起來，再插到尾端

搬之前   [A] ⇄ [B] ⇄ [C] ⇄ [D]        it 指向 B
                 ↑it
① 拆     [A] ⇄———————→ [C] ⇄ [D]       B->prev->next = B->next
② 接     [A] ⇄ [C] ⇄ [D] ⇄ [B]        插到 end() 之前

節點 B 從頭到尾沒有搬家，it 一直指著同一塊記憶體
```

> [!note] `splice(pos, x, i)` 在 `pos == i` 或 `pos == ++i` 時是合法的 no-op
> 所以搬「已經在目標位置」的節點不用先判斷，直接呼叫。省掉 LRU 裡「它是不是已經是最新的」那個特判。

## 四、成員版演算法：不是方便，是泛型版根本不能用

[[STL-Pitfalls]] 第一類講的是「泛型版能用但很慢」。`list` 這裡更絕對——**有些泛型版根本編不過**。

### `std::sort` 對 `list` 是編譯錯誤

```txt
error: no match for ‘operator-’ (operand types are ‘std::_List_iterator<int>’ and ...)
error: no match for ‘operator+’ (operand types are ‘std::_List_iterator<int>’ and ‘int’)
```

introsort 要算中位數、要跳到區間中點，需要 random access iterator。`list` 只有 bidirectional，連 `it + n` 都沒有。所以 `list` 自帶 `sort()` 成員（歸併排序，穩定，只重接指標）。

### 但 `list::sort()` 比倒進 `vector` 排還慢

```txt
排序 100 萬個 int
  l.sort()（成員版歸併）           174.88 ms
  倒進 vector 排完再倒回來          55.29 ms      ← 快 3 倍
  一開始就用 vector（對照組）       46.73 ms
```

「就地排序不用額外記憶體」聽起來很划算，但每次比較都要沿指標跳，全是 cache miss。**如果你發現自己在 `list` 上排序，第一個該問的是「為什麼資料在 `list` 裡」。**

### 完整的成員版清單

| 成員                | 做的事                         | 泛型版的下場                    |
| ------------------- | ------------------------------ | ------------------------------- |
| `l.sort()`          | 歸併排序，穩定                 | `std::sort` **編不過**          |
| `l.merge(other)`    | 合併兩條已排序串列，掏空 other | `std::merge` 要寫到第三個容器   |
| `l.reverse()`       | 反轉                           | `std::reverse` 會逐一交換 value |
| `l.unique()`        | 去除**相鄰**重複，真的刪掉     | `std::unique` 不改 size         |
| `l.remove(v)`       | 刪掉所有等於 v 的，真的刪掉    | `std::remove` 不改 size         |
| `l.remove_if(pred)` | 同上，改用述詞                 | `std::remove_if` 不改 size      |
| `l.splice(...)`     | 搬節點                         | 沒有對應的泛型版                |

> [!important] `list` 的成員版會真的改變 `size()`
> 這正好是 [[STL-Pitfalls]] 第三類的反面。`std::remove` / `std::unique` 只搬元素、回傳新的邏輯結尾，你得自己補 `erase`；**`list` 的同名成員直接把節點釋放掉**：
>
> ```txt
> list{1,1,2,2,2,3,3,4}
>   l.unique()    size 8 -> 4     free 掉 4 個節點
>   l.remove(3)   size 4 -> 3     free 掉 1 個節點
> ```
>
> C++20 之後 `std::erase` / `std::erase_if` 對 `list` 也適用，語意跟 `remove` / `remove_if` 一樣，不用記兩套。

配置行為實測（自訂 allocator 計數）：

```txt
  sort / merge / reverse / splice    alloc=0  free=0     純指標重接
  unique / remove                    alloc=0  free=n     真的釋放被刪的節點
```

> [!warning] `merge` 要求兩邊都已排序，沒排序不報錯也不崩，只給錯答案
>
> ```txt
> list{1,5,3}.merge(list{2,4,6}) = 1 2 4 5 3 6     is_sorted = 0
> ```
>
> 跟 `std::merge`、`lower_bound` 是同一類前提陷阱：**編得過、跑得完、答案是錯的**。

## 五、iterator 失效規則

`list` 的規則簡單到可以背下來：

| 操作                                                | 失效範圍                     |
| --------------------------------------------------- | ---------------------------- |
| `insert` / `push_front` / `push_back` / `emplace_*` | **無**                       |
| `erase` / `pop_front` / `pop_back`                  | **只有被刪的那一個**         |
| `splice`                                            | **無**（被搬的改屬新容器）   |
| `sort` / `reverse` / `merge`                        | **無**（順序變了，指向不變） |
| `clear` / 解構                                      | 全部                         |

`sort` 之後 iterator 仍然有效這點很反直覺——因為它排的是**指標**，節點原地不動。對照 `vector` 的 `sort` 是搬 value，iterator 位置不變但指向的東西全換了。

## 六、`forward_list`：省一根指標，代價很大

|                        | `list`   | `forward_list`         |
| ---------------------- | -------- | ---------------------- |
| 每節點（`int`）        | 24 bytes | 16 bytes               |
| `size()`               | O(1)     | **沒有這個函式**       |
| 反向走訪               | ✓        | ✗                      |
| 在 `it` 之前插入／刪除 | ✓        | ✗，只有 `*_after` 系列 |

```txt
forward_list 要知道長度只能 distance() 走一趟：100 萬個元素 3.64 ms
list 的 size()：n=100 萬與 n=400 萬，100 萬次呼叫都是 0.21 ms   ← O(1)
```

`forward_list` 的所有操作都是 `insert_after` / `erase_after` / `splice_after`，因為單向串列刪節點需要**前驅**。刷題基本用不到，省的那 8 bytes 換不回介面的彆扭。

## 速查：該不該用 `list`

1. **我需要「元素位址永遠不變」嗎？** 不需要 → `vector`。
2. **需要。那位置是誰決定的？** 有序／雜湊決定 → `map` / `unordered_map`（它們的 reference 也穩定）。
3. **由我決定，而且我會從別的結構拿到 iterator 直接動它** → 這才是 `list`。
4. 進到第 3 點之後，搬動一律用 `splice` 單節點版，別用 `erase` + `push_back`。

第 3 點在刷題裡幾乎只有一種形狀：**hash map 存 `list::iterator`，用 O(1) 查找餵給 O(1) 重排。**

## Related Problems

- [[0146-LRU-Cache]] — 這篇所有性質的實戰現場，三種寫法（splice／erase+push_back／手刻雙向串列）並列比較
- [[0460-LFU-Cache]] — 進階版，多一層頻率桶，`splice` 用得更兇
- [[0432-All-Oone-Data-Structure]] — 同款 hash + 雙向串列，串列維護的是計數桶的有序關係
- [[STL-Pitfalls]] — 第一類「泛型演算法 vs 容器成員」的姊妹篇，`list` 是那條規則最極端的例子
- [[0143-Reorder-List]] — 手刻串列的指標操作練習，dummy 節點消除邊界判斷的同一招
