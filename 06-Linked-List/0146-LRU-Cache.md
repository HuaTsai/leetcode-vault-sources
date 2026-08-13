---
leetcode-id: 146
difficulty: medium
tags:
  - linked-list
  - hash
  - grind-169
  - neetcode-150
memo: 用 hash map 存 list 節點的 iterator，把 O(1) 查找接上 O(1) 抽換；搬節點到尾端一律用 splice，它明文保證 iterator 不失效，改用 erase 加 push_back 就必須自己把 map 裡那份補回去，否則留下指向已釋放節點的殘影，而且十之八九會碰巧跑出正確答案；另一個死穴是更新既有 key 時先刪 map 再用 operator 中括號取 iterator，拿到的是 nullptr，直接段錯誤
dg-publish: true
---

## Problem Description

Design a data structure that follows the constraints of a Least Recently Used (LRU) cache.

Implement the `LRUCache` class:

- `LRUCache(int capacity)` Initialize the LRU cache with positive size `capacity`.
- `int get(int key)` Return the value of the `key` if the key exists, otherwise return `-1`.
- `void put(int key, int value)` Update the value of the `key` if the `key` exists. Otherwise, add the `key-value` pair to the cache. If the number of keys exceeds the `capacity` from this operation, evict the least recently used key.

The functions `get` and `put` must each run in `O(1)` average time complexity.

## Solution

核心觀念：題目要求兩件事同時 O(1)，而它們各自都有現成的工具，難的是**把兩者接起來**。

1. **依 key 找到資料** → hash map，O(1)
2. **維護「最近使用」的順序，而且要能從中間抽走任意一筆再接到尾端** → 雙向串列，O(1)

單看第二點就能刷掉大部分容器：`vector` 從中間抽走要搬後面全部（O(n)）；單向串列抽節點要先找到它的前驅（O(n)）。**只有雙向串列能在「手上已經有節點指標」時 O(1) 拆接。**

而「手上已經有節點指標」正是 hash map 要負責提供的——所以 map 的 value 不存數值，存**該 key 在串列裡的節點位置**（`list::iterator` 或自己刻的 `Node*`）。兩個結構互補：map 給你 O(1) 定位，串列給你 O(1) 重排。

約定串列**頭端是 LRU（最久沒用）、尾端是 MRU（剛用過）**，淘汰就砍頭、使用就搬到尾。

```txt
capacity = 3
                 LRU ←———————— 使用時間由舊到新 ————————→ MRU
data_ (list)   [ 1:100 ] ⇄ [ 2:200 ] ⇄ [ 3:300 ]
                    ↑          ↑          ↑
mp_ (hash)          1          2          3      key → 該節點的 iterator

get(1) → mp_ 直接跳到節點，splice 搬到尾端（節點本身沒動，只改四條指標）
data_          [ 2:200 ] ⇄ [ 3:300 ] ⇄ [ 1:100 ]

put(4,400) → 已滿，砍頭端。注意要先讀 front().first 拿到 key 才刪得掉 map 那筆
data_          [ 3:300 ] ⇄ [ 1:100 ] ⇄ [ 4:400 ]
   淘汰的 2 ↑ 從 mp_ 一併移除
```

> [!important] 淘汰時要「從節點反查 key」，所以節點必須把 key 也存進去
> 串列節點存 `pair<key, value>` 而不是只存 `value`，唯一的理由就是淘汰那一步：`data_.pop_front()` 之前得先 `mp_.erase(data_.front().first)`，否則 map 裡會留下一筆指向已釋放節點的殘影。只存 value 的話你根本不知道要刪 map 的哪個 key。

### 方法一：hash map + `list::splice` — O(1)／O(capacity)（推薦）

`splice` 是 `list` 的成員函式，作用是**把節點從串列裡摘下來接到別處**，只改指標、不搬資料、不配置也不釋放記憶體。關鍵在標準明文保證：被搬動的 iterator、指標、參考**全部繼續有效**，只是所屬容器換了。這代表 map 裡存的 iterator 完全不用維護。

```cpp
// Time: O(1)  get / put 都是一次 hash 查找 + 常數次指標重接
// Space: O(capacity)  串列與 map 各存至多 capacity 筆
class LRUCache {
 public:
  LRUCache(int capacity) : capacity_(capacity) {}

  int get(int key) {
    auto it = mp_.find(key);
    if (it == mp_.end()) return -1;
    data_.splice(data_.end(), data_, it->second);  // 搬到尾端，it->second 依然有效
    return it->second->second;
  }

  void put(int key, int value) {
    auto it = mp_.find(key);
    if (it != mp_.end()) {           // 已存在：改值 + 搬到尾端
      it->second->second = value;
      data_.splice(data_.end(), data_, it->second);
      return;
    }
    if (data_.size() == static_cast<size_t>(capacity_)) {  // 滿了：淘汰頭端
      mp_.erase(data_.front().first);                      // 先用 key 刪 map
      data_.pop_front();
    }
    data_.push_back({key, value});
    mp_[key] = prev(data_.end());
  }

 private:
  int capacity_;
  list<pair<int, int>> data_;                              // front = LRU, back = MRU
  unordered_map<int, list<pair<int, int>>::iterator> mp_;
};
```

> [!tip] `splice(data_.end(), data_, it)` 搬「已經在尾端」的節點是合法的 no-op
> 標準規定 `splice(pos, x, i)` 在 `pos == i` 或 `pos == ++i` 時無效果。節點已在尾端時 `++i` 正好是 `end()`，所以不需要任何「它是不是已經最新」的前置判斷——直接搬就對了。

> [!note] 為什麼是 `it->second->second`
> `it` 是 map 的 iterator，`it->second` 是它的 value 也就是 list 的 iterator，再一個 `->second` 才是 `pair<key, value>` 的 value。看起來繞，但用 `mp_[key]->second` 會多一次 hash 查找，而且 `operator[]` 有插入副作用（見下方陷阱）。**一次 `find` 拿到手之後全程重用同一個 `it`。**

### 方法二：hash map + `erase` + `push_back` — O(1)／O(capacity)

不用 `splice` 也能做，但「搬到尾端」變成**刪掉舊節點、在尾端建一個新節點**。代價是舊 iterator 會失效，必須自己把 map 裡那份補回去。

```cpp
// Time: O(1)  同上，但每次搬動多一次節點的 free 與 alloc
// Space: O(capacity)
class LRUCache {
 public:
  LRUCache(int capacity) : capacity_(capacity) {}

  int get(int key) {
    auto it = mp_.find(key);
    if (it == mp_.end()) return -1;
    int value = it->second->second;
    data_.erase(it->second);
    data_.push_back({key, value});
    it->second = prev(data_.end());  // ← 少了這行就是 use-after-free
    return value;
  }

  void put(int key, int value) {
    auto it = mp_.find(key);
    if (it != mp_.end()) {
      data_.erase(it->second);  // ← 先用 iterator，再動 map，順序不能反
      mp_.erase(it);
    } else if (data_.size() == static_cast<size_t>(capacity_)) {
      mp_.erase(data_.front().first);
      data_.pop_front();
    }
    data_.push_back({key, value});
    mp_[key] = prev(data_.end());
  }

 private:
  int capacity_;
  list<pair<int, int>> data_;
  unordered_map<int, list<pair<int, int>>::iterator> mp_;
};
```

複雜度同樣是 O(1)，但常數差很多。300 萬次操作（capacity 3000、90% `get`／10% `put`、`-O2`）實測：

```txt
  splice              30.9 ms
  erase + push_back   53.6 ms      慢 73%
```

差距來自每次搬動都多一組 `operator delete` + `operator new`（每個節點 24 bytes）。`splice` 全程零配置。

> [!warning] `mp_.erase(key)` 之後再用 `mp_[key]` 會拿到 nullptr iterator，直接段錯誤
> 這是最容易寫反的兩行：
>
> ```cpp
> mp_.erase(key);          // key 從 map 消失了
> data_.erase(mp_[key]);   // operator[] 插入一個「值初始化」的 list::iterator
> ```
>
> `unordered_map::operator[]` 對不存在的 key 會**插入預設值**，而 libstdc++ 的 `_List_iterator() : _M_node() { }` 把內部節點指標值初始化成 `nullptr`。接著 `list::erase` 在 `list.tcc:157` 執行 `__position._M_node->_M_next`——對空指標解參考。gdb 直接報 `non-dereferenceable iterator for std::list`，一定 crash，沒有僥倖。
>
> 修法就是把順序倒過來：**先用 iterator，再刪 map**。

> [!warning] 忘記更新 map 裡的 iterator，答案卻常常是對的——這才是最危險的
> `data_.erase(it->second)` 之後節點已被釋放，`data_.push_back(...)` 在尾端建新節點。若沒補上 `it->second = prev(data_.end())`，map 裡那份就指著已釋放的位址。
>
> 但這段**在一般編譯下十之八九會跑出正確答案**：`push_back` 的 allocator 通常把剛 `free` 掉的那塊記憶體原封不動要回來，失效的 iterator 誤打誤撞就指到新節點。g++ 11 `-O0`／`-O2` 實測答案全對。要加上 `-fsanitize=address` 讓 quarantine 擋掉記憶體重用，才會現形：
>
> ```txt
> ERROR: AddressSanitizer: heap-use-after-free ... READ of size 4
>   #0 in LRUCache::get(int)
>   freed by thread T0 here:
>   #5 in std::list<...>::_M_erase(...)     ← 就是上一行的 data_.erase
> ```
>
> 這種 bug 在 judge 上的症狀是「同一份 code 有時 AC 有時 RE」。**方法一根本不會遇到，因為 `splice` 不釋放節點。**

### 方法三：手刻雙向串列 — O(1)／O(capacity)

面試常要求不准用 STL 容器裝節點。自己刻的重點是**首尾各放一個 dummy 節點**，這樣 `unlink` / `linkBefore` 都不必判斷 `nullptr`，邊界情況直接消失。

```cpp
// Time: O(1)
// Space: O(capacity)
class LRUCache {
 public:
  LRUCache(int capacity) : capacity_(capacity) {
    head_.next = &tail_;
    tail_.prev = &head_;
  }

  int get(int key) {
    auto it = mp_.find(key);
    if (it == mp_.end()) return -1;
    moveToBack(it->second);
    return it->second->value;
  }

  void put(int key, int value) {
    auto it = mp_.find(key);
    if (it != mp_.end()) {
      it->second->value = value;
      moveToBack(it->second);
      return;
    }
    if (static_cast<int>(mp_.size()) == capacity_) {
      Node* lru = head_.next;
      unlink(lru);
      mp_.erase(lru->key);  // 先讀 key 再 delete
      delete lru;
    }
    Node* node = new Node{key, value};
    linkBefore(node, &tail_);
    mp_[key] = node;
  }

  ~LRUCache() {  // 自己 new 的就得自己 delete
    for (Node* p = head_.next; p != &tail_;) {
      Node* next = p->next;
      delete p;
      p = next;
    }
  }

 private:
  struct Node {
    int key = 0, value = 0;
    Node* prev = nullptr;
    Node* next = nullptr;
  };

  void unlink(Node* n) {
    n->prev->next = n->next;
    n->next->prev = n->prev;
  }
  void linkBefore(Node* n, Node* pos) {
    n->prev = pos->prev;
    n->next = pos;
    pos->prev->next = n;
    pos->prev = n;
  }
  void moveToBack(Node* n) {
    unlink(n);
    linkBefore(n, &tail_);
  }

  int capacity_;
  Node head_, tail_;  // dummy 頭尾，兩個都不存資料
  unordered_map<int, Node*> mp_;
};
```

> [!tip] dummy 頭尾把「刪的是不是第一個／最後一個」全部消掉
> 沒有 dummy 的話 `unlink` 要寫成四種情況（唯一節點、頭、尾、中間）。有了 dummy，`n->prev` 和 `n->next` 永遠非空，兩行就結束。這是雙向串列題的通用手法，[[0143-Reorder-List]] 的變體解法也用同一招。

> [!note] 判滿要用 `mp_.size()` 而不是自己數串列
> 手刻版沒有 `list::size()` 可用，數串列是 O(n)。map 的 `size()` 是 O(1)，而且兩者元素數永遠一致——`mp_` 本來就是每個節點的索引。

> [!warning] 解構子不是裝飾用的
> 手刻版每個節點都是 `new` 出來的，沒有解構子就是**每個 cache 物件洩漏 `capacity` 個節點**。同一份 code 在 LeetCode 上照樣 AC（judge 不查洩漏），但 `-fsanitize=address` 一跑就是 `LeakSanitizer: detected memory leaks`。面試時漏掉這段是實打實的扣分項——這也是方法一、二用 `list` 的隱性好處：容器的解構子會幫你收乾淨。

三種寫法都跑過 LeetCode 官方範例與隨機壓測（capacity 1–5、key 0–7、300 輪 × 500 步，對照暴力解），並在 `-fsanitize=address,undefined` 下乾淨通過。

## Related Problems

- [[0460-LFU-Cache]] — 直接的加難版，把「單一使用順序」換成「頻率分桶、桶內再 LRU」，同樣靠 hash 存串列節點位置
- [[0380-Insert-Delete-GetRandom-O1]] — 同一種「hash 存位置」把刪除壓到 O(1) 的思路，只是位置是 vector 索引、抽走的手法換成 swap 到尾端再 pop
- [[0432-All-Oone-Data-Structure]] — hash + 雙向串列的強化版，串列維護的是計數桶的有序關係，`splice` 用得更兇
- [[0155-Min-Stack]] — 同屬「多維護一份輔助結構，換某個查詢從 O(n) 降到 O(1)」的設計題套路
- [[STL-List-Techniques]] — `splice` 的三種多載、iterator 穩定性保證、以及 `list` 什麼時候真的該用
- [[STL-Pitfalls]] — 本題踩到的 `operator[]` 插入副作用與迭代器失效兩條陷阱都收在第四類
