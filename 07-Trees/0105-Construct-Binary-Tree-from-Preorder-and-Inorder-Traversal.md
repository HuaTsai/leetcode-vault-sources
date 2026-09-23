---
leetcode-id: 105
difficulty: medium
tags:
  - array
  - hash
  - tree
  - binary-tree
  - grind-169
  - neetcode-150
memo: preorder[0] 是根、拿去 inorder 定位切出左右子樹，兩側切片長度必須一致；但那個線性查找其實是多餘的——中序游標遲早會走過同一段，改用邊界值當遞迴參數就能兩個陣列各掃一次，O(n) 且比任何查表法都快
dg-publish: true
---

## Problem Description

Given two integer arrays `preorder` and `inorder` where `preorder` is the preorder traversal of a binary tree and `inorder` is the inorder traversal of the same tree, construct and return the binary tree.

Constraints:

- preorder and inorder consist of unique values.

## Solution

核心觀念：兩個序列各給一半資訊——**preorder 說「誰是根」，inorder 說「根把節點分成哪兩群」**。前者只看得到第一格，後者要先知道根才有用，所以每層都是「取根 → 定位 → 切兩半 → 遞迴」。

```txt
preorder = [3, 9, 20, 15, 7]      根排在整段最前面
             ↑
inorder  = [9, 3, 15, 20, 7]      根把中序切成左右兩群
            └┬┘ ↑  └───┬───┘
             左  根     右
           lsz=1       rsz=3

回頭切 preorder：根之後緊接整個左子樹，再來才是右子樹
preorder = [3 | 9 | 20, 15, 7]
            根  左1    右3         ← 長度由 inorder 決定，順序由 preorder 決定
```

> [!important] inorder 算長度，preorder 定順序
> 不變量只有一條：**同一層的兩個區段必須描述同一群節點，長度相等**。左子樹在 inorder 佔 `[0, lsz)`、在 preorder 佔 `[1, 1+lsz)`；右子樹在 inorder 佔 `[pivot+1, n)`、在 preorder 佔 `[1+lsz, n)`。**preorder 那側的起點只跟 `lsz` 有關，跟 `pivot` 無關**——`pivot` 是 inorder 的座標，搬過去只剩「長度」還有效。兩者剛好同值（`lsz == pivot`）是巧合，寫成 `pivot + 1` 才看得出自己在哪個座標系。

### 方法一：分治 ＋ `span` 切片 — O(n²) 最壞／O(h)（推薦先掌握）

把「這棵子樹的兩段」直接寫進參數。`span` 是最合身的型別：非擁有、零拷貝、`subspan` 就是切片。代價是每層線性掃 inorder 找根。

```cpp
// Time: O(n²) 最壞（左斜樹每層掃到底），隨機樹約 O(n log n)
// Space: O(h)  只有遞迴堆疊
class Solution {
 public:
  TreeNode* buildSubTree(span<const int> preorder, span<const int> inorder) {
    if (preorder.empty()) {
      return nullptr;
    }
    int val = preorder[0];
    int pivot = find(inorder.begin(), inorder.end(), val) - inorder.begin();
    int lsz = pivot;
    int rsz = inorder.size() - pivot - 1;
    return new TreeNode(
      val,
      buildSubTree(preorder.subspan(1, lsz), inorder.subspan(0, lsz)),
      buildSubTree(preorder.subspan(1 + lsz, rsz), inorder.subspan(pivot + 1, rsz))
    );
  }

  TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
    return buildSubTree(preorder, inorder);
  }
};
```

> [!warning] `span` 的兩個真陷阱
>
> - **兩個參數同型別，傳反了不會有任何抗議**，而且會建出一棵節點數正確、形狀合理的樹——因為「把 inorder 當前序」本身也是合法輸入，只是描述了另一棵樹。實測傳反：20000 棵隨機樹裡 17137 棵錯，其餘是左右對稱剛好矇對。**「建得出一棵樹」不能當正確性證據**。定錨點只有一條：根來自 `preorder[0]`，pivot 去 `inorder` 找。
> - **`subspan` 越界是 UB 而且安靜**。`subspan(3, 4)` 對 size 5 的 span：一般編譯 exit 0 印垃圾；`-fsanitize=address` 要等你真的讀到那格才報；只有 **`-D_GLIBCXX_ASSERTIONS` 會在 `subspan` 那一行當場 abort**。本機編譯固定加上它。

> [!tip] 找 pivot 用 `std::find`，不是 `ranges::find`
> 三種寫法在左斜鏈（掃描主導）的指令數：手寫 `for` 663M、`ranges::find` 531M、`std::find` **308M**。libstdc++ 的 `std::find` 對 random-access iterator 有 4 路展開的特化（`trip_count = (last - first) >> 2`，每圈比 4 次），`ranges::find` **沒有**，它是樸素的單步迴圈。這裡新介面比舊介面慢。
>
> 另外 `find(...) - begin()` 和 `distance(begin(), ...)` 組語完全相同（random-access 會 dispatch 到相減），純看可讀性；`std::distance` 的價值在非 random-access 的泛型場合，這裡沒東西可保。

### 方法二：雙游標遞迴，根本不查找 — O(n)／O(h)

方法一那個掃描是多餘的：中序走訪的定義就是整個遞迴會由左到右掃過 `inorder` 一次，所以**你掃的那段正是左子樹的中序區段，而左子樹的遞迴等一下會再走一遍**。pivot 的位置＝「左子樹建完後中序游標停的地方」，你是在預先算一個遞迴馬上會免費給你的答案。

把問題從「根在 inorder 的哪裡」換成「這段中序走完了沒」，後者只要跟父層傳下來的**邊界值**比一次：

```cpp
// Time: O(n)   兩個游標都只往前走，每個元素各碰一次
// Space: O(h)  只有遞迴堆疊，沒有任何輔助結構
class Solution {
 public:
  TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
    pre = preorder.data();
    in = inorder.data();
    n = preorder.size();
    return build(nullptr);
  }

 private:
  const int *pre, *in;
  int n, p = 0, i = 0;

  TreeNode* build(const int* stop) {
    if (i == n || (stop && in[i] == *stop)) {
      return nullptr;
    }
    TreeNode* node = new TreeNode(pre[p++]);
    node->left = build(&node->val);  // 左子樹的中序在「自己」之前結束
    ++i;                             // 消費掉 node 自己
    node->right = build(stop);       // 右子樹跟父層在同一處結束
    return node;
  }
};
```

> [!tip] `build(stop)` 的契約：建出中序區段在 `stop` 這個值之前結束的子樹
> 左子樹的界線是 `node` 自己，右子樹沿用父層的界線，最外層沒有界線。**`stop` 用 `const int*` 而不是 `INT_MIN` 哨兵**是刻意的——[[0098-Validate-Binary-Search-Tree]] 就是栽在拿值域極值代表「無界」上，這裡 `nullptr` 讓「沒有界線」有自己的表示，節點值等於 `INT_MIN` 也不會誤判。代價是可讀性：`stop` 是一個邊界**值**而不是範圍，不像 `subspan` 會自我說明，而且兩個陣列退回成 member 游標，簽名不再描述自己的輸入。

### 方法三：迭代，用 stack 把中序走訪反過來跑 — O(n)／O(n)

方法二把遞迴堆疊攤開來就是這個。`i` 是中序游標，stack 裡永遠是一條從根往下的左鏈；`st.top()->val == inorder[i]` 代表「這條左鏈走到底了」，該往回彈到第一個還缺右子的祖先。

```cpp
// Time: O(n)   每個節點進出 stack 各一次，i 單調前進
// Space: O(n)  stack 最壞裝下整條左鏈
class Solution {
 public:
  TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
    TreeNode* root = new TreeNode(preorder[0]);
    stack<TreeNode*> st;
    st.push(root);
    int i = 0;
    for (int k = 1; k < (int)preorder.size(); ++k) {
      TreeNode* node = new TreeNode(preorder[k]);
      if (st.top()->val != inorder[i]) {
        st.top()->left = node;
      } else {
        TreeNode* parent = nullptr;
        while (!st.empty() && st.top()->val == inorder[i]) {
          parent = st.top();
          st.pop();
          ++i;
        }
        parent->right = node;
      }
      st.push(node);
    }
    return root;
  }
};
```

> [!warning] 內層 while 結束後要掛在 `parent` 上，不是 `st.top()`
> `parent` 是**最後一個被彈掉的**節點，也就是那個缺右子的祖先；`st.top()` 已經是它的父輩了。這是這個寫法最常見的 bug。

> [!note] 實測：查表法是在為一個不該存在的問題付錢
> n=2000、`g++ -O2`、cachegrind 指令數（重複 50 次）：
>
> | 寫法                          | 隨機樹     | 左斜鏈       | 右斜鏈     |
> | ----------------------------- | ---------- | ------------ | ---------- |
> | 方法二（雙游標，不查找）      | **30.2M**  | **29.4M**    | **28.6M**  |
> | 方法三（迭代，也不查找）      | 32.0M      | 29.2M        | 31.6M      |
> | 方法一（`std::find`）         | 34.9M      | 307.7M       | 32.2M      |
> | 方法一（手寫迴圈）            | 33.6M      | 663.4M       | 30.6M      |
> | 方法一改成 6001 格陣列查表    | 52.1M      | 46.7M        | 47.2M      |
> | 方法一改成 `unordered_map`    | 110.8M     | 109.0M       | 108.3M     |
>
> - **把線性查找換成 O(1) 查表反而更慢**：陣列版在隨機樹上比 O(n²) 的方法一慢 49%，`unordered_map` 版慢 3.2 倍。它們讓多餘的問題變便宜，而不是消滅它。真正的 O(n) 解（方法二、三）不需要任何輔助結構，也因此最快。相關討論見 [[Micro-Optimization-Myths]]。
> - **方法一的 O(n²) 只在左斜樹發作**：左斜鏈 307.7M 是右斜鏈 32.2M 的 **9.6 倍**，互為鏡像的兩棵樹差這麼多，因為右斜樹的根永遠就在 `inorder[0]`。
> - 本題 n ≤ 3000，方法一在官方測資（隨機樹）上其實是最快的一群，AC 沒有問題。**先問「這個查找是必要的嗎」，再問「怎麼讓查找變快」**，才是這題真正的教學點。

> [!warning] 值必須互異，這不是題目在客氣
> 允許重複的話，`preorder=[1,1]`／`inorder=[1,1]` 同時對應「根有左子」和「根有右子」兩棵樹，pivot 無從決定。同理 preorder ＋ postorder 也還原不出唯一解——inorder 的不可取代之處在於它是唯一能把節點分成「左邊那群／右邊那群」的序列。

## Related Problems

- [[0106-Construct-Binary-Tree-from-Inorder-and-Postorder-Traversal]] — 完全鏡像：根取 postorder 最後一格，游標由後往前，而且遞迴必須**先建右子樹**，否則游標會錯位
- [[0889-Construct-Binary-Tree-from-Preorder-and-Postorder-Traversal]] — 對照組：少了 inorder 就分不出左右邊界，只有一個子節點時答案不唯一，所以那題只要求回傳任一解
- [[1008-Construct-Binary-Search-Tree-from-Preorder-Traversal]] — BST 的 inorder 就是排序後的 preorder，一個陣列就夠；本題的第二個陣列補的正是 BST 免費提供的資訊
- [[0098-Validate-Binary-Search-Tree]] — 方法二的 `nullptr` 哨兵跟那題是同一課；另一面是中序序列作為樹的指紋，那題用它的遞增性、本題用它的位置
- [[Tree-Traversal-Iterative]] — 方法三是那篇「模板二：中序」的逆運算，把走訪時的 push／pop 反過來當建樹指令
