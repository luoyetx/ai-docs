# Trie 数据结构：从基础到高级变种全解析

## 一、什么是 Trie

Trie（读作 /ˈtraɪ/，来源于 Re**trie**val）又称**前缀树**或**字典树**，是一种多叉树形数据结构，用于高效地存储和检索字符串集合。核心思想是：**共享公共前缀，将查找时间从与集合大小相关，变为仅与字符串长度相关。**

Edward Fredkin 在 1960 年将这一结构正式命名为 Trie。此后 60 多年间，Trie 衍生出了大量变种，覆盖从嵌入式系统到内存数据库的广泛场景。

### 1.1 基本性质

- 根节点不包含字符，除根以外每个节点代表一个字符
- 从根到某个节点的路径拼接起来，就是该节点对应的字符串前缀
- 每个节点的子节点代表的字符互不相同
- 标记为"终止"的节点表示从根到此处的路径构成一个完整字符串

```
存储 {"apple", "app", "apply", "bat", "bar"}:

              (root)
             /      \
            a        b
            |        |
            p        a
            |       / \
            p      t   r
           /|\     |   |
          l  [*]  [*] [*]
         / \
        e   y
        |   |
       [*] [*]

[*] = is_end 标记
```

### 1.2 与其他数据结构对比

| 操作 | 哈希表 | 平衡 BST | Trie |
|------|--------|----------|------|
| 精确查找 | O(m) 平均 | O(m · log n) | O(m) |
| 插入 | O(m) 平均 | O(m · log n) | O(m) |
| 前缀搜索 | O(n · m) | O(m · log n + k) | O(m + k) |
| 有序遍历 | O(n · log n) | O(n) | O(n) |
| 最长前缀匹配 | 不原生支持 | O(m · log n) | O(m) |
| 空间 | O(n · m) | O(n · m) | 取决于前缀共享度 |

> m = 字符串长度，n = 集合大小，k = 匹配结果数

Trie 的核心优势在于：**查找时间只与字符串长度相关，与集合大小无关**。哈希表虽然平均 O(m)，但存在哈希冲突和扩容的开销；Trie 没有这些问题，而且天然支持前缀查询和字典序遍历。

---

## 二、基础 Trie（Standard Trie）

### 2.1 节点结构

最直接的实现：每个节点维护一个固定大小的子节点数组。假设字符集为小写英文字母（σ = 26）：

```cpp
struct TrieNode {
    TrieNode* children[26];   // 按字符索引的子节点指针
    bool is_end;              // 是否为某个单词的结尾
    int count;                // 可选：以此为前缀的单词计数

    TrieNode() : is_end(false), count(0) {
        memset(children, 0, sizeof(children));
    }
};
```

如果需要支持全 ASCII 或 Unicode，可以把数组大小改为 128 / 256，或者用 `unordered_map<char, TrieNode*>` 动态存储。

### 2.2 基本操作

**插入 — O(m)：**

```cpp
void insert(TrieNode* root, const string& word) {
    TrieNode* node = root;
    for (char c : word) {
        int idx = c - 'a';
        if (!node->children[idx]) {
            node->children[idx] = new TrieNode();
        }
        node = node->children[idx];
        node->count++;  // 可选：维护前缀计数
    }
    node->is_end = true;
}
```

**精确查找 — O(m)：**

```cpp
bool search(TrieNode* root, const string& word) {
    TrieNode* node = root;
    for (char c : word) {
        int idx = c - 'a';
        if (!node->children[idx]) return false;
        node = node->children[idx];
    }
    return node->is_end;
}
```

**前缀查询 — O(m)：**

```cpp
bool starts_with(TrieNode* root, const string& prefix) {
    TrieNode* node = root;
    for (char c : prefix) {
        int idx = c - 'a';
        if (!node->children[idx]) return false;
        node = node->children[idx];
    }
    return true;  // 不需要检查 is_end
}
```

**前缀枚举 — O(m + k)：**

```cpp
void collect(TrieNode* node, string& current, vector<string>& results) {
    if (!node) return;
    if (node->is_end) results.push_back(current);
    for (int i = 0; i < 26; i++) {
        if (node->children[i]) {
            current.push_back('a' + i);
            collect(node->children[i], current, results);
            current.pop_back();
        }
    }
}

vector<string> autocomplete(TrieNode* root, const string& prefix) {
    TrieNode* node = root;
    for (char c : prefix) {
        int idx = c - 'a';
        if (!node->children[idx]) return {};
        node = node->children[idx];
    }
    vector<string> results;
    string current = prefix;
    collect(node, current, results);
    return results;
}
```

**删除 — O(m)：**

删除需要注意不能破坏其他字符串的路径。递归实现最清晰：

```cpp
// 返回 true 表示当前节点可以被释放
bool remove(TrieNode* node, const string& word, int depth) {
    if (!node) return false;

    if (depth == (int)word.size()) {
        if (!node->is_end) return false;  // 字符串不存在
        node->is_end = false;
        // 如果没有任何子节点，可以释放
        for (int i = 0; i < 26; i++) {
            if (node->children[i]) return false;
        }
        return true;
    }

    int idx = word[depth] - 'a';
    if (remove(node->children[idx], word, depth + 1)) {
        delete node->children[idx];
        node->children[idx] = nullptr;

        // 如果当前节点也不是终止且没有其他子节点，可以继续向上释放
        if (!node->is_end) {
            for (int i = 0; i < 26; i++) {
                if (node->children[i]) return false;
            }
            return true;
        }
    }
    return false;
}
```

### 2.3 空间分析

基础 Trie 最大的问题是**空间浪费**。每个节点分配 σ 个指针，但大部分为空：

```
假设存储 10 万个英文单词，平均长度 8：
- 节点总数 ≈ 60 万（大量前缀共享后）
- 每个节点: 26 × 8 字节（指针） + 元数据 ≈ 216 字节
- 总空间 ≈ 124 MB

如果支持全 ASCII (σ = 256):
- 每个节点: 256 × 8 = 2048 字节
- 总空间 > 1 GB
```

这在很多场景下不可接受，因此催生了各种优化变种。

### 2.4 子节点存储策略对比

除了固定数组，还有其他方式存储子节点：

| 策略 | 查找子节点 | 空间/节点 | 适用场景 |
|------|-----------|----------|----------|
| 固定数组 | O(1) | O(σ) | σ 小且密集 |
| 排序数组 | O(log σ) | O(实际子节点数) | σ 大但子节点少 |
| 链表 | O(σ) 最坏 | O(实际子节点数) | 空间敏感 |
| 哈希表 | O(1) 平均 | O(实际子节点数) | 通用 |
| Bitmap + 紧凑数组 | O(1) + popcount | O(实际子节点数) | HAMT 等高级结构 |

### 2.5 适用场景

- **自动补全**：输入法、IDE 代码补全、搜索框建议
- **拼写检查**：快速判断单词是否在词典中
- **IP 地址前缀匹配**（简化场景）
- **词频统计**：在 `is_end` 节点附加计数器
- **教学与竞赛**：实现直观，适合理解前缀树思想

!!! note "何时不该用基础 Trie"
    如果你的场景只需要精确查找（不需要前缀查询），哈希表几乎总是更好的选择。Trie 的优势在于前缀相关的操作，如果用不到，就是在为一个用不上的特性付出空间代价。

---

## 三、压缩 Trie（Radix Tree / Patricia Trie）

### 3.1 核心思想

基础 Trie 中，如果一条路径上的节点只有一个子节点且非终止节点，那么这条"链"可以被**压缩为一条边**。压缩后的边上存储的不再是单个字符，而是一个字符串片段（label）。

```
基础 Trie:                    Radix Tree:
    (root)                      (root)
      |                        /     \
      t                    "te"      "inn"
      |                    /   \        |
      e                 "a"   "st"    [*]
     / \                 |      |
    a   s              [*]    [*]
    |   |
   [*]  t
        |
       [*]

存储: "tea", "test", "inn"
基础 Trie 节点数: 9          Radix Tree 节点数: 6
```

压缩规则很简单：**只要节点只有一个子节点且不是终止节点，就和子节点合并。**

### 3.2 历史与术语

这个领域的术语比较混乱，需要厘清：

- **Patricia Trie**（1968, Donald R. Morrison）：原始论文指的是二叉 Trie（按 bit 分支），名字来自 "Practical Algorithm To Retrieve Information Coded In Alphanumeric"
- **Radix Tree**：泛指基于 radix 分支的压缩 Trie。radix = 2^k：
    - k = 1 → 按 bit 分支（经典 Patricia）
    - k = 4 → 按 nibble（半字节）分支
    - k = 8 → 按 byte 分支（最常见）
- **Compact Trie / Compressed Trie**：泛称，等同于 Radix Tree

在现代用法中，这三个术语经常混用，通常都指按字节分支的压缩 Trie。

### 3.3 节点结构

```cpp
struct RadixNode {
    string edge_label;              // 边上的字符串片段
    unordered_map<char, RadixNode*> children;  // 按 label 首字符索引
    bool is_end;
    void* value;                    // 可选：关联值

    RadixNode() : is_end(false), value(nullptr) {}
};
```

!!! tip "实际实现中的 label 存储"
    高性能实现通常不在边上存储字符串拷贝，而是存储**原始 key 的偏移和长度**（类似 `string_view`），避免内存分配和拷贝。Redis 的 Rax 实现就是这样做的。

### 3.4 插入算法详解

Radix Tree 的插入比基础 Trie 复杂，因为需要处理**边的分裂（split）**。有三种情况：

```
情况 1: 没有匹配的边 → 直接创建新叶节点

  node                    node
   |          插入 "xyz"    |  \
 "abc"       ────────→  "abc" "xyz"
   |                      |     |
  [*]                    [*]   [*]


情况 2: 边完全匹配 → 沿着树继续向下

  node                    node
   |          插入 "abcxyz"  |
 "abc"       ────────→    "abc"
   |                        |
  [*]                     (继续处理 "xyz")


情况 3: 部分匹配 → 分裂边

  node                      node
   |          插入 "abxyz"    |
 "abcde"     ────────→     "ab"        ← 新分裂节点
   |                       /    \
  [*]                   "cde"  "xyz"    ← 旧后缀 + 新后缀
                          |      |
                         [*]    [*]
```

完整实现：

```cpp
void insert(RadixNode* root, const string& key) {
    RadixNode* node = root;
    string remaining = key;

    while (!remaining.empty()) {
        // 查找以 remaining[0] 开头的边
        auto it = node->children.find(remaining[0]);

        if (it == node->children.end()) {
            // 情况 1: 无匹配边，直接挂新叶节点
            auto* leaf = new RadixNode();
            leaf->edge_label = remaining;
            leaf->is_end = true;
            node->children[remaining[0]] = leaf;
            return;
        }

        RadixNode* child = it->second;
        string& label = child->edge_label;

        // 计算公共前缀长度
        int common = 0;
        while (common < (int)remaining.size() && common < (int)label.size()
               && remaining[common] == label[common]) {
            common++;
        }

        if (common == (int)label.size()) {
            // 情况 2: 边完全被消费，继续向下
            remaining = remaining.substr(common);
            node = child;
            if (remaining.empty()) {
                child->is_end = true;  // key 正好在此节点结束
                return;
            }
            continue;
        }

        // 情况 3: 部分匹配，需要分裂
        auto* split = new RadixNode();
        split->edge_label = label.substr(0, common);
        node->children[remaining[0]] = split;

        // 旧子节点下沉
        child->edge_label = label.substr(common);
        split->children[child->edge_label[0]] = child;

        // 新 key 的剩余部分
        string suffix = remaining.substr(common);
        if (suffix.empty()) {
            split->is_end = true;
        } else {
            auto* new_leaf = new RadixNode();
            new_leaf->edge_label = suffix;
            new_leaf->is_end = true;
            split->children[suffix[0]] = new_leaf;
        }
        return;
    }
    node->is_end = true;
}
```

### 3.5 查找算法

```cpp
bool search(RadixNode* root, const string& key) {
    RadixNode* node = root;
    string remaining = key;

    while (!remaining.empty()) {
        auto it = node->children.find(remaining[0]);
        if (it == node->children.end()) return false;

        RadixNode* child = it->second;
        const string& label = child->edge_label;

        // 检查 remaining 是否以 label 开头
        if (remaining.size() < label.size() ||
            remaining.substr(0, label.size()) != label) {
            return false;
        }

        remaining = remaining.substr(label.size());
        node = child;
    }
    return node->is_end;
}
```

### 3.6 最长前缀匹配

Radix Tree 天然擅长**最长前缀匹配**（Longest Prefix Match, LPM）——在所有匹配的前缀中找到最长的那个。这正是 IP 路由查找的核心操作：

```cpp
string longest_prefix_match(RadixNode* root, const string& key) {
    RadixNode* node = root;
    string remaining = key;
    string last_match;
    string current;

    while (!remaining.empty()) {
        auto it = node->children.find(remaining[0]);
        if (it == node->children.end()) break;

        RadixNode* child = it->second;
        const string& label = child->edge_label;

        if (remaining.size() < label.size()) {
            // 边标签比剩余 key 长，检查部分匹配
            // 如果 remaining 是 label 的前缀，不算匹配
            break;
        }

        if (remaining.substr(0, label.size()) != label) break;

        current += label;
        remaining = remaining.substr(label.size());
        node = child;

        if (node->is_end) {
            last_match = current;
        }
    }
    return last_match;
}
```

### 3.7 空间与时间

| 属性 | 基础 Trie | Radix Tree |
|------|----------|------------|
| 节点数上限 | O(n · m) | O(n) |
| 内部节点数 | 最多 n·m | **≤ n − 1**（n 为键数） |
| 查找 | O(m) | O(m) |
| 插入 | O(m) | O(m)（含分裂） |
| 删除 | O(m) | O(m)（含合并） |
| 实现复杂度 | 简单 | 中等 |

关键指标：**Radix Tree 的内部节点数不超过 n − 1**。这意味着节点总数为 O(n)，与键的长度无关，空间效率大幅提升。直觉上：每次插入最多增加一个叶节点和一个分裂产生的内部节点。

### 3.8 实际应用

**Linux 内核 — 页缓存索引（历史）：**

Linux 内核长期使用 Radix Tree 将文件页偏移映射到 `struct page`。每个 `address_space` 持有一棵 Radix Tree，key 是页索引（`pgoff_t`），按 6 bits 一级分支（radix = 64）。内核 4.20 之后被 **XArray** 取代，但 XArray 底层仍然是 Radix Tree，只是 API 更简洁。

**Redis Rax — Stream ID 索引：**

Redis 的 Rax 是一个紧凑的 Radix Tree 实现，用于 Stream 数据类型的 ID 索引。Rax 的节点将 key 字节、子节点指针、关联值全部打包在一块连续内存中，非常 cache 友好：

```
[header | key_bytes | padding | child_ptrs | value_ptr]
```

**HTTP 路由器：**

Go 的 `httprouter`、Rust 的 `actix-web` 路由器都基于 Radix Tree。URL path 作为 key，handler 作为 value。路径参数（如 `/users/:id`）通过特殊边处理。Radix Tree 在这里的优势是：路由查找时间只与 URL 深度相关，与注册的路由总数无关。

**IP 路由表：**

网络路由器用 Patricia Trie（按 bit 分支）做最长前缀匹配。给定目标 IP 地址，找到路由表中匹配的最长网络前缀，确定下一跳。Linux 内核的 FIB（Forwarding Information Base）就是基于这一结构的（实际实现是 LC-Trie，一种层压缩的变种）。

---

## 四、Ternary Search Tree（三叉搜索树 / TST）

### 4.1 设计动机

基础 Trie 在每个节点用数组索引子节点，查找子节点 O(1) 但空间浪费大（大部分槽位为空）。如果改用哈希表或链表，空间减少了但增加了常数因子。

**TST** 是 Jon Bentley 和 Robert Sedgewick 在 1997 年提出的折中方案。它融合了 Trie 和 BST 的思想：每个节点存储一个字符，有三个子指针——**小于、等于、大于**：

```
查找 "cute":

         c          ← 'c' == 'c'，走 mid
       / | \
      b  u  h       ← 'u' == 'u'，走 mid
         |
         t          ← 't' == 't'，走 mid
        / \
       s   x        ← 'e' < 't'? 不对...
```

更准确地说，TST 的遍历规则是：

1. 比较当前字符 `key[d]` 与节点字符 `node.c`
2. 如果 `key[d] < node.c` → 走 left（同一层，不前进）
3. 如果 `key[d] > node.c` → 走 right（同一层，不前进）
4. 如果 `key[d] == node.c` → 走 mid（**前进到下一个字符 d+1**）

### 4.2 节点结构

```cpp
struct TSTNode {
    char c;          // 本节点代表的字符
    bool is_end;     // 是否为某个词的结尾
    TSTNode* left;   // 字符 < c
    TSTNode* mid;    // 字符 == c，前进到 key 的下一个字符
    TSTNode* right;  // 字符 > c

    TSTNode(char ch) : c(ch), is_end(false),
                        left(nullptr), mid(nullptr), right(nullptr) {}
};
```

每个节点只有 **3 个指针**，远少于基础 Trie 的 26 或 256 个。

### 4.3 完整操作

**插入：**

```cpp
TSTNode* insert(TSTNode* node, const string& word, int depth) {
    char ch = word[depth];

    if (!node) {
        node = new TSTNode(ch);
    }

    if (ch < node->c) {
        node->left = insert(node->left, word, depth);
    } else if (ch > node->c) {
        node->right = insert(node->right, word, depth);
    } else {
        // 匹配，前进到下一个字符
        if (depth + 1 < (int)word.size()) {
            node->mid = insert(node->mid, word, depth + 1);
        } else {
            node->is_end = true;
        }
    }
    return node;
}
```

**查找：**

```cpp
bool search(TSTNode* node, const string& word, int depth) {
    if (!node) return false;
    char ch = word[depth];

    if (ch < node->c) {
        return search(node->left, word, depth);
    } else if (ch > node->c) {
        return search(node->right, word, depth);
    } else {
        if (depth + 1 == (int)word.size()) {
            return node->is_end;
        }
        return search(node->mid, word, depth + 1);
    }
}
```

**近似匹配（Hamming 距离 ≤ d）：**

TST 非常适合近似匹配。以下是搜索与 pattern 至多有 `dist` 个字符不同的所有词：

```cpp
void near_search(TSTNode* node, const string& pattern, int depth,
                 int dist, string& current, vector<string>& results) {
    if (!node || dist < 0) return;

    char ch = (depth < (int)pattern.size()) ? pattern[depth] : 0;

    // 如果还有"预算"，探索 left（字符不匹配，但不消耗距离——因为我们还没匹配到这个字符）
    if (ch < node->c || dist > 0) {
        near_search(node->left, pattern, depth, dist, current, results);
    }

    // mid: 匹配或替换
    if (depth < (int)pattern.size()) {
        current.push_back(node->c);
        int new_dist = (ch == node->c) ? dist : dist - 1;

        if (depth + 1 == (int)pattern.size() && node->is_end && new_dist >= 0) {
            results.push_back(current);
        }
        near_search(node->mid, pattern, depth + 1, new_dist, current, results);
        current.pop_back();
    }

    if (ch > node->c || dist > 0) {
        near_search(node->right, pattern, depth, dist, current, results);
    }
}
```

### 4.4 性能特点

| 属性 | 值 |
|------|-----|
| 查找 | O(m · log σ) 最坏，O(m) 平均 |
| 插入 | O(m · log σ) 最坏 |
| 空间/节点 | 3 个指针 + 1 字符 ≈ 25 字节（vs Trie 的 200+ 字节） |
| 前缀搜索 | 天然支持 |
| 近似搜索 | 天然支持且高效 |
| 有序遍历 | 天然支持（中序遍历 left/mid/right） |

!!! warning "插入顺序的影响"
    TST 的性能高度依赖插入顺序。如果按字典序插入，left/right 子树会退化为链表，查找退化为 O(m · σ)。解决方案：

    - 随机化插入顺序
    - 将 left/right 子树维护为平衡 BST（如红黑树）
    - 先排序，然后按中位数分治的顺序插入

### 4.5 适用场景

- **拼写纠错 / 模糊搜索**：TST 的近似匹配能力是其最大特色。支持 Hamming 距离、通配符匹配，甚至 edit distance 搜索
- **自动补全**：前缀搜索 + 有序遍历，可以快速返回按字典序排列的候选项
- **字典和词库**：空间效率远好于基础 Trie，功能远多于哈希表
- **不适合**：需要确定性 O(m) 查找的场景（如路由表）

---

## 五、DAFSA / DAWG（有向无环词图）

### 5.1 从 Trie 到 DAWG

Trie 通过共享前缀来节省空间，但**不共享后缀**。如果存储 `{"testing", "resting", "fishing", "dishing"}`，后缀 `"ting"` 和 `"shing"` 各出现两次，在 Trie 中会被存储两份。

**DAWG（Directed Acyclic Word Graph）** 也叫 **DAFSA（Deterministic Acyclic Finite State Automaton）**，在 Trie 的基础上进一步合并**后缀相同且后续结构完全等价**的子树：

```
Trie:                                    DAWG:
      (root)                              (root)
      /    \                              /    \
     t      r                            t      r
     |      |                            |      |
     e      e                            e      e
     |      |                             \    /
     s      s                              s
     |      |                              |
     t      t            ──────→           t
     |      |                              |
     i      i                              i
     |      |                              |
     n      n                              n
     |      |                              |
     g      g                              g
     |      |                              |
    [*]    [*]                            [*]

"testing" 和 "resting"             后缀 "sting" 共享一份
```

注意：合并的条件不仅仅是"后缀字符串相同"，而是**整个子树结构等价**（包括所有分支和终止标记）。

### 5.2 等价关系的定义

两个节点 u、v 在以下条件下等价（可合并）：

1. `u.is_end == v.is_end`
2. u 和 v 有相同的出边字符集
3. 对于每个相同的出边字符 c，`u.child(c)` 和 `v.child(c)` 等价（递归定义）

这本质上是 DFA 最小化中的 Myhill-Nerode 等价关系。DAWG 就是 Trie 对应的最小 DFA。

### 5.3 构建方法

**方法一：离线构建（Trie → DAWG）**

1. 先构建完整 Trie
2. 自底向上对子树做签名（哈希），签名相同的子树合并
3. 时间 O(n · m)，空间需要同时持有 Trie 和签名表

**方法二：增量构建（Revuz 算法，1992）**

要求输入按字典序排列。核心思想是：维护一个"right language"注册表，每次新词插入后，从最后一个分歧点开始检查是否有可合并的节点。

```python
class DAWG:
    def __init__(self):
        self.root = Node()
        self.register = {}        # 签名 → 节点
        self.previous_word = ""

    def insert(self, word):
        assert word > self.previous_word  # 必须字典序

        # 找到与上一个词的公共前缀
        common = common_prefix_length(word, self.previous_word)

        # 从公共前缀之后的节点开始，尝试合并（freeze）
        self._freeze_suffix(common)

        # 添加新词的独有后缀
        node = self._traverse(word[:common])
        for ch in word[common:]:
            child = Node()
            node.children[ch] = child
            node = child
        node.is_end = True

        self.previous_word = word

    def _freeze_suffix(self, from_depth):
        """将 previous_word[from_depth:] 路径上的节点注册或合并"""
        # 从最深处向 from_depth 回溯
        # 对每个节点计算签名，查注册表
        # 如果注册表中有等价节点，替换（共享）
        # 否则注册当前节点
        ...
```

增量构建的优势是内存效率高——不需要同时持有完整 Trie。

### 5.4 空间压缩效果

| 数据集 | Trie 节点数 | DAWG 节点数 | 压缩比 |
|--------|-----------|------------|--------|
| 英文 Scrabble 词典（~27万词） | ~120万 | ~6万 | 20:1 |
| 英文拼写词典（~10万词） | ~50万 | ~3万 | 17:1 |
| 法语词典（~30万词） | ~150万 | ~5万 | 30:1 |

自然语言词典的压缩效果特别好，因为自然语言有大量重复的后缀（-tion, -ing, -ment, -ness 等）。

### 5.5 操作

DAWG 支持：

- **精确查找** — O(m)：与 Trie 完全相同
- **前缀判断** — O(m)：从根开始匹配前缀
- **字典序遍历** — O(n)：DFS 可以按字典序输出所有词

DAWG **不擅长**：

- 前缀枚举（列出所有以给定前缀开头的词）：由于后缀共享，同一个节点可能通过不同路径到达，枚举会产生重复或遗漏
- 动态插入/删除：合并后的结构牵一发而动全身，修改成本极高
- 关联值存储：节点共享后，不同词的"同一个终止节点"无法区分

### 5.6 适用场景

- **拼写检查词典**：静态词典的存在性判断，空间极致压缩
- **Scrabble / 填字游戏**：需要快速判断是否为合法单词
- **自然语言处理中的形态学分析**：判断词根、词缀的合法组合
- **压缩存储大规模字符串集合**：如 DNS 域名列表、URL 去重

!!! note "DAWG vs GADDAG"
    在 Scrabble 游戏中，常用 GADDAG（Gordon's Adapted DAWG）代替 DAWG。GADDAG 将每个词存储为多个变换形式（如 "CAT" 存储为 "C>AT", "AC>T", "TAC>"），从而支持在任意位置向两个方向扩展，这是棋盘游戏中常见的查询模式。

---

## 六、Double-Array Trie

### 6.1 设计思想

由 Jun-ichi Aoe 在 1989 年提出。核心洞察：Trie 的树结构可以用**两个一维整型数组**完整表示，实现接近指针 Trie 的查找速度和远超指针 Trie 的空间效率。

两个数组的含义：

- `base[s]`：状态 s 的基地址，用于计算子节点的位置
- `check[t]`：状态 t 的父状态校验，确认转移的合法性

从状态 s 经字符 c 转移到状态 t 的规则：

```
t = base[s] + c
valid iff check[t] == s
```

### 6.2 详细图解

存储 `{"ac", "ace", "ad", "b"}` 的过程：

```
字符编码: # (结束符) = 0, a = 1, b = 2, c = 3, d = 4, e = 5

第一步: 插入 "ac"
  需要转移: root --a--> s1 --c--> s2 --#--> (终止)

  为 root 找 base: base[0] 需要使得 base[0]+1 和 base[0]+2 都空闲
  设 base[0] = 1

  root --a--> t = base[0] + 1 = 2, set check[2] = 0
  s1=2, 为 s1 找 base: base[2] 需要使得 base[2]+3 空闲
  设 base[2] = 1
  s1 --c--> t = base[2] + 3 = 4, set check[4] = 2
  s2=4, base[4] = 1, s2 --#--> t = base[4] + 0 = 1, check[1] = 4

最终结果:
index:  0    1    2    3    4    5    6    7    8
base:  [ 1, -1,  1,  -,   1,  -1,  1,  -1,  -]
check: [ -,  4,  0,  0,   2,   2,  4,   6,  -]

查找 "ace":
  root(0) → base[0]+a = 1+1 = 2, check[2]=0 ✓ → 状态 2
  s(2) → base[2]+c = 1+3 = 4, check[4]=2 ✓ → 状态 4
  s(4) → base[4]+e = 1+5 = 6, check[6]=4 ✓ → 状态 6
  s(6) → base[6]+# = 1+0 = 1? check[1]=4 ✗ → 不是合法终止?
```

!!! note "终止标记的处理"
    Double-Array Trie 有两种方式标记终止状态：

    1. **加结束符**：为每个词末尾追加特殊字符 `#`，转移到一个终止状态。终止状态的 base 值设为负数
    2. **负 base 值**：如果状态 s 是终止状态，`base[s]` 取负值（编码关联值的索引），同时仍可有子节点转移

    实际实现中方法 2 更常用，空间更紧凑。

### 6.3 构建算法

构建的核心难点是**找到合适的 base 值**，使所有子节点的目标位置不与已有状态冲突：

```python
def build(trie_node, state, base_arr, check_arr):
    children = get_children(trie_node)  # [(char, child_node), ...]
    if not children:
        return

    # 找到一个 base 值，使得所有 base + c 位置都空闲
    codes = [encode(ch) for ch, _ in children]
    base_val = find_valid_base(codes, check_arr)
    base_arr[state] = base_val

    # 设置所有子节点
    for ch, child_node in children:
        t = base_val + encode(ch)
        check_arr[t] = state
        build(child_node, t, base_arr, check_arr)

def find_valid_base(codes, check_arr):
    """找到最小的 base 值，使 base + c 对所有 c 都空闲"""
    base = 1
    while True:
        if all(check_arr[base + c] == EMPTY for c in codes):
            return base
        base += 1
```

朴素构建是 O(n · σ · A) 的（A 为数组大小），可以通过维护**空闲位置链表**优化到接近 O(n · σ)。

### 6.4 冲突解决与重定位

当已有状态的子节点与新状态冲突时，需要**重定位**（relocate）已有的某个状态：

```
冲突场景:
  新状态 s_new 想把子节点放在位置 t，但 check[t] != EMPTY

解决方案:
  1. 找到占据位置 t 的状态 s_old (check[t] = parent(s_old))
  2. 比较 s_new 和 parent(s_old) 的子节点数量
  3. 重定位子节点更少的那个:
     - 为其找到新的 base 值
     - 搬迁所有子节点
     - 更新所有子节点的 check 值
     - 递归更新子节点的子节点的 check 值
```

重定位是 Double-Array 构建中最昂贵的操作，这也是为什么 Double-Array Trie 通常用于**离线构建、在线查找**的场景。

### 6.5 查找性能分析

```
查找 "apple":
  操作序列:
    t = base[0] + 'a'     → 1次数组访问 + 1次比较
    t = base[t] + 'p'     → 1次数组访问 + 1次比较
    t = base[t] + 'p'     → 1次数组访问 + 1次比较
    t = base[t] + 'l'     → 1次数组访问 + 1次比较
    t = base[t] + 'e'     → 1次数组访问 + 1次比较

  总计: m 次数组索引 + m 次整数比较
  无分支、无哈希计算、无指针追踪
  → 极其 cache 友好
```

### 6.6 与动态 Double-Array

原始 Double-Array 不擅长动态更新，但后续有改进：

- **cedar**（Naoki Yoshinaga, 2014）：支持动态插入和删除的 Double-Array Trie，通过维护空闲位置的双向链表和更智能的 base 值选择策略，将动态操作性能提升了数量级
- **Darts-clone**：Darts 的改进版，使用 32-bit 单数组编码（将 base 和 check 压缩到一个 uint32 中），空间减半

### 6.7 适用场景

- **中日韩分词**：MeCab（日文）、Kuromoji（Java 日文分词）、jieba（中文）等分词器的核心数据结构。CJK 语言字符集大（σ > 3000），用基础 Trie 空间爆炸，但 Double-Array 可以紧凑表示
- **Aho-Corasick 多模式匹配**：AC 自动机的 goto 函数可以用 Double-Array 编码，避免巨大的转移矩阵
- **输入法词库**：存储拼音到候选词的映射，词库可达百万级
- **静态字典查找**：任何需要高速查找、离线构建的场景

!!! tip "实用工具"
    - **Darts**（C++）：最经典的 Double-Array 实现
    - **cedar**（C++）：支持动态操作的高性能版本
    - **python-datrie**（Python）：libdatrie 的 Python 绑定
    - **Darts-clone**（C++）：空间减半的变种

---

## 七、Adaptive Radix Tree（ART）

### 7.1 问题背景

在内存数据库场景中，索引结构需要同时满足：

1. 查找速度快（接近或超过哈希表）
2. 支持范围查询和有序遍历（哈希表做不到）
3. 空间占用合理
4. 并发友好

传统选择是 B+ 树，但 B+ 树是为磁盘 I/O 优化的，在纯内存场景下并非最优。传统 Trie 查找 O(m) 但空间爆炸。

**ART**（由 Viktor Leis, Alfons Kemper, Thomas Neumann 在 2013 年于 TU Munich 提出）的核心创新：**根据子节点数量动态选择不同的节点类型**，在空间和速度之间自适应平衡。

### 7.2 四种节点类型

```
┌─────────┬──────────┬───────────────────────────────────┬────────────┐
│ 类型     │ 子节点数  │ 内部结构                           │ 查找方式    │
├─────────┼──────────┼───────────────────────────────────┼────────────┤
│ Node4   │ 1-4      │ 4-byte key[] + 4 ptrs[]           │ 线性扫描    │
│ Node16  │ 5-16     │ 16-byte key[] + 16 ptrs[]         │ SIMD 比较   │
│ Node48  │ 17-48    │ 256-byte index[] + 48 ptrs[]      │ 间接索引    │
│ Node256 │ 49-256   │ 256 ptrs[] (与经典 Trie 相同)       │ 直接索引    │
└─────────┴──────────┴───────────────────────────────────┴────────────┘
```

详细内存布局：

```
Node4 (空间: ~52 字节):
┌──────────────────────────────────┐
│ type: uint8                       │
│ count: uint8                      │
│ prefix[8]: uint8                  │
│ prefix_len: uint32                │
│ keys[4]:  [ a | c | _ | _ ]      │  ← 有序，线性扫描
│ child[4]: [ptr|ptr| _ | _ ]      │
└──────────────────────────────────┘

Node16 (空间: ~272 字节):
┌──────────────────────────────────┐
│ type, count, prefix...           │
│ keys[16]: [a|b|c|d|e|f|g|h|...] │  ← 有序，SIMD 并行比较
│ child[16]:[*|*|*|*|*|*|*|*|...] │
└──────────────────────────────────┘
  查找: 将目标字节广播为 16 字节向量，
        与 keys 做 SIMD 比较，一条指令完成

Node48 (空间: ~656 字节):
┌──────────────────────────────────┐
│ type, count, prefix...           │
│ index[256]: [_|_|1|_|0|_|2|...] │  ← 全字节索引，O(1) 定位
│ child[48]:  [ptr|ptr|ptr|...]    │  ← 紧凑指针数组
└──────────────────────────────────┘
  查找: slot = index[key_byte]
        if slot != EMPTY → child[slot]

Node256 (空间: ~2064 字节):
┌──────────────────────────────────┐
│ type, count, prefix...           │
│ child[256]: [ptr|ptr|...|ptr]    │  ← 直接索引，与经典 Trie 相同
└──────────────────────────────────┘
  查找: child[key_byte]
```

### 7.3 SIMD 加速（Node16）

Node16 的查找利用了 SSE/NEON SIMD 指令，可以在**一条指令**中完成 16 路比较：

```cpp
// x86 SSE2 实现
int find_child_node16(Node16* node, uint8_t key_byte) {
    // 将 key_byte 广播到 16 字节向量
    __m128i key_vec = _mm_set1_epi8(key_byte);

    // 与 keys[16] 做并行比较
    __m128i cmp = _mm_cmpeq_epi8(key_vec,
                    _mm_loadu_si128((__m128i*)node->keys));

    // 将比较结果转为 bitmask
    int mask = _mm_movemask_epi8(cmp);

    // mask 中第一个 1 的位置就是匹配的 slot
    if (mask) {
        return __builtin_ctz(mask);  // count trailing zeros
    }
    return -1;  // 未找到
}
```

在 ARM 上使用 NEON 指令可以实现类似效果。

### 7.4 节点增长与收缩

当子节点数超过当前节点容量时，**升级**到更大的节点类型；删除导致子节点数过少时，**降级**：

```
Node4 ←→ Node16 ←→ Node48 ←→ Node256
      4→5    16→17      48→49          ← 升级阈值
      ←4     ←12        ←37           ← 降级阈值（有滞后，避免抖动）
```

升级和降级是原子的——分配新节点、拷贝数据、替换指针。降级阈值故意低于升级阈值，防止在边界处反复升降。

### 7.5 路径压缩

ART 结合了 Radix Tree 的路径压缩。论文提出两种策略：

**悲观策略（Pessimistic）：**

在节点中存储压缩路径的前 8 个字节。如果压缩路径超过 8 字节，需要回查完整 key：

```cpp
struct ArtNode {
    uint8_t type;
    uint8_t prefix_len;     // 压缩前缀的完整长度
    uint8_t prefix[8];      // 最多存 8 字节前缀
    // ... 子节点相关字段
};

// 查找时:
// 1. 比较 min(prefix_len, 8) 个字节
// 2. 如果 prefix_len > 8，需要从叶节点取完整 key 来验证
```

**乐观策略（Optimistic）：**

只存储 prefix_len，不存储前缀内容。查找时直接跳过 prefix_len 个字节，最终在叶节点与完整 key 比较验证。这种方式节省空间但在不匹配时可能浪费查找时间。

### 7.6 性能对比

论文中的实测数据（单线程，随机 key）：

| 结构 | 查找 (M ops/s) | 插入 (M ops/s) | 空间 (bytes/key) |
|------|---------------|---------------|-----------------|
| 红黑树 | ~5 | ~4 | ~72 |
| 哈希表 | ~20 | ~15 | ~80 |
| B+ 树 | ~12 | ~8 | ~40 |
| ART | ~25 | ~20 | ~30-50 |

ART 在查找、插入速度上超过哈希表（得益于更好的 cache 局部性），同时支持范围查询——这是哈希表做不到的。

### 7.7 并发 ART（CART / ART-OLC）

ART 的后续工作引入了并发版本：

- **ART-OLC**（Optimistic Lock Coupling）：每个节点一个版本号锁，读操作乐观无锁，写操作加锁。类似 B-link Tree 的 lock coupling
- **ART-ROWEX**（Read-Optimized Write EXclusion）：读操作完全无锁，写操作用 epoch-based reclamation 管理内存

### 7.8 适用场景

- **内存数据库索引**：HyPer（ART 的发源地）、DuckDB 都使用 ART 作为默认索引
- **键值存储**：适合 key 为可变长字节串的场景（如字符串 key、复合 key）
- **替代 B+ 树**：在纯内存场景下，ART 通常优于 B+ 树（B+ 树的优势在于磁盘 I/O，在内存中反而是劣势）
- **替代哈希索引**：当需要范围查询（range scan）时，ART 是比哈希更好的选择
- **时序数据库**：时间戳作为 key，ART 的字节级分支天然适合

!!! warning "ART 的局限"
    ART 的空间效率高度依赖 key 的分布。如果 key 是随机长字符串（几乎没有公共前缀），ART 退化为每个 key 一条长路径，空间和查找时间都不如哈希表。ART 最擅长的是：key 有公共前缀且分布不太稀疏的场景（如 URL、文件路径、IP 地址、时间戳）。

---

## 八、HAT-Trie

### 8.1 设计思想

由 Nikolas Askitis 和 Ranjan Sinha 在 2007 年提出。核心观察：Trie 的上层（靠近根）具有高扇出，指针开销合理；下层（靠近叶）扇出低，指针开销浪费。

HAT-Trie 的解决方案：**上层用 Trie 节点，下层用哈希桶**。"HAT" 代表 Hash Array Trie。

```
上层: 标准 Trie 节点（高扇出，空间利用率好）
          (root)
         /  |  \
        a   b   c
       / \  |
      p   r a    ...
      |   |
─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ 分界线 ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
      |   |
下层: 哈希桶（紧凑存储后缀）
  [hash bucket]   [hash bucket]
  "ple" → v1      "ray" → v2
  "ply" → v4      "rch" → v5
  "oint"→ v7
```

### 8.2 两种容器类型

HAT-Trie 定义了两种叶容器：

**Array Hash Container：**

将后缀字符串紧凑地打包在连续内存块中：

```
┌─────────────────────────────────────────────────┐
│ len₁ | suffix₁ | val₁ | len₂ | suffix₂ | val₂ |...
└─────────────────────────────────────────────────┘

查找方式: 线性扫描（容器小的时候很快，因为 cache line 内）
```

**Hash Table Container：**

当容器内元素较多时，使用开放寻址哈希表：

```
┌────────────────────────────────────────┐
│ slot[0] | slot[1] | ... | slot[cap-1]  │
└────────────────────────────────────────┘
每个 slot 存储: (hash, suffix_offset, value)
后缀字符串另存在紧凑的字符串池中
```

### 8.3 桶分裂（Burst）

当容器中的元素数超过阈值（通常 16K-32K 字节），执行分裂：

```
分裂前:                        分裂后:
  parent                        parent
    |                             |
[container]                   [Trie 节点]
"apple" "apply"              /    |    \
"apt" "arch"               "p"  "r"   ...
                           / \    |
                        "pl" "t" "ch"
                         |    |    |
                      [cont] [cont] [cont]
                      "e""y" ""     ""
```

分裂过程：

1. 取容器中所有后缀的第一个字符
2. 创建一个新的 Trie 内部节点
3. 按第一个字符分组，每组的后缀（去掉首字符后）放入新的子容器
4. 如果某个分组太大，可以递归分裂

### 8.4 性能实测

Askitis 和 Sinha 的论文实测（英文词典 + URL 数据集）：

| 操作 | HAT-Trie | 基础 Trie | std::map | 哈希表 |
|------|----------|----------|----------|--------|
| 查找 | 最快 | 2-3x 慢 | 5-10x 慢 | 接近 |
| 插入 | 最快 | 2-3x 慢 | 5-10x 慢 | 接近 |
| 内存 | 最小 | 5-10x 大 | 2-3x 大 | 1.5-2x 大 |
| 有序遍历 | 支持 | 支持 | 支持 | 不支持 |
| 前缀搜索 | 支持 | 支持 | 不支持 | 不支持 |

HAT-Trie 之所以比纯 Trie 更快，是因为：下层容器的数据紧凑存储在连续内存中，一次 cache line 加载可以覆盖多个后缀的比较。而纯 Trie 每走一步都是一次指针追踪，几乎每步都 cache miss。

### 8.5 适用场景

- **内存中的大规模字符串字典**：当你需要 Trie 的前缀查询能力，但空间预算有限
- **全文索引的词典部分**：搜索引擎中 term → posting list 的映射
- **大规模 URL 去重**：爬虫场景，需要高速判断 URL 是否已访问
- **任何需要"哈希表的速度 + Trie 的前缀查询"的场景**

!!! tip "实用工具"
    - **hat-trie**（C++，Tessil/hat-trie）：高质量的现代 C++ 实现，支持 C++11 容器接口
    - **hat-trie**（Go）：Go 实现

---

## 九、Burst Trie

### 9.1 设计思想

由 Heinz, Zobel, Williams 在 2002 年提出，比 HAT-Trie 更早。结构上与 HAT-Trie 类似——也是上层 Trie + 下层容器——但下层容器使用**排序结构**（有序链表或排序数组）而非哈希表。

```
          (Access Trie)
            (root)
           /  |  \
          a   b   c
          |   |   |
        ┌───┐ ┌───┐ ┌───┐
        │ b │ │ a │ │ a │   ← Burst Container
        │ c │ │ o │ │ u │     (排序链表/数组)
        │ d │ │ y │ │ t │
        └───┘ └───┘ └───┘
```

### 9.2 Burst 策略

当容器满时，"burst" 操作将其转化为一个 Trie 节点加多个子容器。Burst Trie 的论文提出了几种 burst 策略：

**按容量（Limit Burst）：** 容器中元素数超过阈值 C 时 burst。C 的典型值为 256-8192。

**按比例（Ratio Burst）：** 当最频繁子字符的比例超过某个阈值时 burst。这样可以让高扇出的容器更快分裂。

**按趋势（Trend Burst）：** 监控容器的增长趋势，快速增长的容器优先 burst。

### 9.3 与 HAT-Trie 的区别

| 属性 | Burst Trie | HAT-Trie |
|------|-----------|----------|
| 容器类型 | 排序链表/数组 | 哈希表/紧凑数组 |
| 有序遍历 | O(n)，天然有序 | 需要排序 |
| 容器内查找 | O(log k) 或 O(k) | O(1) 平均 |
| 提出时间 | 2002 | 2007 |
| 空间效率 | 好 | 更好 |
| 查找速度 | 快 | 更快 |

### 9.4 适用场景

- **字符串排序**：Burst Sort（基于 Burst Trie 的排序算法）在大规模字符串排序中比标准快排快 2-4 倍
- **需要严格有序遍历的场景**：如果你需要频繁按字典序遍历所有 key，Burst Trie 比 HAT-Trie 更合适
- **文本索引构建**：构建倒排索引时，需要对 term 排序并去重

---

## 十、DAFSA / Minimal DAFSA（最小化有限自动机）

> 见第五章 DAWG。此处补充其作为有限自动机的视角。

DAFSA 的正式名称是**最小确定性有限自动机**（Minimal DFA for a finite language）。从自动机理论角度看：

- 一个字符串集合定义了一个**有限语言** L
- 存在一个**唯一的最小 DFA** 识别 L（Myhill-Nerode 定理）
- DAFSA / DAWG 就是这个最小 DFA

这意味着 DAFSA 是理论上最紧凑的精确匹配数据结构——不可能有比它更小的（在节点数意义上）确定性自动机来识别同一个字符串集合。

---

## 十一、Succinct Trie

### 11.1 信息论视角

一棵有 n 个节点的有序树，可能的形状数为第 n 个 Catalan 数 C(n) ≈ 4^n / (n^1.5 · √π)。因此，表示一棵树至少需要 log₂(C(n)) ≈ 2n bits。

传统指针表示使用 O(n · w) bits（w = 指针宽度，通常 64），是下界的 O(w) 倍。

**Succinct 数据结构**的目标是使用 **2n + o(n) bits**——仅比信息论下界多一个低阶项——同时保持 O(1) 的导航操作。

### 11.2 LOUDS 编码

LOUDS（Level-Ordered Unary Degree Sequence）是最简单的 Succinct Tree 编码，由 Jacobson 在 1989 年提出：

1. 按 BFS 顺序遍历树
2. 对每个节点，输出其度数的一元编码：d 个 `1` 后跟一个 `0`
3. 在整个序列前加 `10`（虚拟超级根）

```
树:
        a (度 3)
      / | \
     b  c  d (度 0, 2, 1)
       / \   \
      e   f   g (度 0, 0, 0)

BFS 序: a, b, c, d, e, f, g

LOUDS:
  超级根: 10
  a: 1110    (度 3)
  b: 0       (度 0)
  c: 110     (度 2)
  d: 10      (度 1)
  e: 0       (度 0)
  f: 0       (度 0)
  g: 0       (度 0)

完整位串: 10 1110 0 110 10 0 0 0
```

总共 2n + 1 bits（n 个 `1` + n+1 个 `0`）。

### 11.3 rank 和 select 操作

LOUDS 编码配合两个位运算原语实现 O(1) 导航：

- `rank1(i)`：位串前 i 位中 `1` 的个数
- `select1(j)`：第 j 个 `1` 的位置
- `rank0(i)` / `select0(j)`：类似，针对 `0`

这两个操作可以通过预计算的辅助结构在 O(1) 时间内完成，辅助结构只需要 o(n) 额外空间。

**树导航：**

```
节点编号按 BFS 序从 1 开始。LOUDS 位串中，节点 i 对应:
  - 子节点范围的起始: select0(i) + 1
  - 第 k 个子节点: 位串中 select0(i) + k 处为 1，
                    子节点编号 = rank1(select0(i) + k)
  - 父节点: select1(rank0(pos))，其中 pos 是节点在位串中的位置
```

### 11.4 LOUDS-Trie

将 LOUDS 应用于 Trie，额外需要存储每条边的字符标签和终止标记：

```
空间分析:
  树结构:     2n bits (LOUDS)
  字符标签:   n × 8 bits (每条边一个字节)
  终止标记:   n bits (每个节点一个 bit)
  rank/select: o(n) bits (辅助结构)
  总计:       ~11n bits

对比:
  指针 Trie (σ=26): ~1700 bits/node
  指针 Trie (σ=256): ~16400 bits/node
  Succinct Trie:     ~11 bits/node

  压缩比: 150:1 到 1500:1
```

### 11.5 查找实现

```
查找 "cat":
  从根开始:
  1. 找根的子节点范围 → 扫描字符标签找 'c' → 得到子节点 x
  2. 找 x 的子节点范围 → 扫描字符标签找 'a' → 得到子节点 y
  3. 找 y 的子节点范围 → 扫描字符标签找 't' → 得到子节点 z
  4. 检查 z 的终止标记 → is_end[z] == 1 → 找到

每步涉及: 1 次 select0 + 线性扫描（子节点数通常很少）+ 1 次 rank1
总计: O(m × σ_avg)，其中 σ_avg 是平均扇出
```

### 11.6 LOUDS-Dense 和 LOUDS-Sparse（SuRF）

**SuRF**（Succinct Range Filter，由 Huanchen Zhang 等人在 2018 年提出）将 Succinct Trie 进一步优化，分为两层：

**LOUDS-Dense（上层，靠近根）：**

每个节点用 256-bit bitmap 表示有哪些子边，加一个 256-bit bitmap 表示有哪些子边对应终止节点。适合高扇出。

```
节点 [a, c, f 有子边]:
  D-HasChild: 10100100 00000000 ... (256 bits)
  D-IsPrefixKey: 1 bit
  D-Labels: 不需要（bitmap 直接编码）
```

**LOUDS-Sparse（下层，靠近叶）：**

每个节点只存实际存在的子边字符（类似原始 LOUDS-Trie）。适合低扇出。

两层的分界根据空间效率自动选择。

### 11.7 适用场景

- **LSM-Tree 的布隆过滤器替代品**（SuRF）：支持点查询和范围查询，而布隆过滤器只支持点查询。RocksDB 实验性支持 SuRF
- **只读字典的极致压缩**：当你有百万级字符串集合，只需要查找和前缀匹配，Succinct Trie 的空间效率无出其右
- **嵌入式系统 / IoT**：内存极其有限的场景
- **网络数据平面**：如 Name Data Networking（NDN）的 FIB 表

!!! warning "Succinct Trie 的代价"
    极致的空间效率是以查找速度为代价的。Succinct Trie 的查找涉及大量位操作（rank/select），常数因子远大于指针 Trie 或 Double-Array。如果空间不是瓶颈，其他变种通常更快。

!!! tip "实用工具"
    - **marisa-trie**（C++/Python）：Matching Algorithm with Recursively Implemented StorAge，基于 LOUDS 的高压缩 Trie
    - **louds-rs**（Rust）：Succinct LOUDS 实现
    - **SuRF**（C++）：论文配套实现，可集成到存储引擎

---

## 十二、x-fast Trie 与 y-fast Trie

### 12.1 问题背景

这两种结构解决的问题不是字符串存储，而是**整数集合**上的前驱/后继查询：

> 给定一个整数集合 S（值域 [0, U)），快速回答：对于查询值 q，S 中小于 q 的最大元素（前驱）和大于 q 的最小元素（后继）是什么？

这在计算几何、数据库索引、网络路由等领域非常重要。

### 12.2 x-fast Trie

由 Dan Willard 在 1983 年提出。结构：

- 构建一棵**高度为 log U 的完整二叉 Trie**（按整数的二进制位索引）
- 只保留实际存在的路径上的节点
- **每一层**维护一个**哈希表**，存储该层所有实际存在的节点
- 叶子层用**双向链表**按顺序串联

```
存储 {1, 4, 5, 7}，U = 8 (w = 3 bits):

Level 0: {ε}                           ← 根（哈希表 H₀）
Level 1: {0, 1}                        ← 哈希表 H₁
Level 2: {00, 10, 11}                  ← 哈希表 H₂
Level 3: {001, 100, 101, 111}          ← 叶子 + 双向链表

二叉 Trie 视图:
              ε
            /   \
           0     1
          /     / \
         00    10  11
          \   / \    \
          001 100 101 111
           ↔   ↔   ↔   ↔    ← 叶子间双向链表
```

**前驱/后继查找算法：**

1. 在各层哈希表上做**二分搜索**，找到查询值 q 的最长匹配前缀
2. 最长前缀对应的节点有一个指针指向其子树中最近的叶子
3. 通过叶子链表找到前驱/后继

二分搜索的对象是"层号"（0 到 log U），每次检查中间层的哈希表中是否存在 q 的前缀。因此前驱/后继查询时间为 **O(log log U)**——是对 log U 做二分。

| 操作 | 时间 | 说明 |
|------|------|------|
| 查找 | O(1) | 直接查叶子层哈希表 |
| 前驱/后继 | O(log log U) | 在层号上二分搜索 |
| 插入/删除 | O(log U) | 需要更新每层哈希表 |
| 空间 | O(n · log U) | 每个元素在每层有一个节点 |

### 12.3 y-fast Trie

x-fast Trie 的问题：空间 O(n · log U)，当 U 很大（如 2^64）时空间爆炸。

y-fast Trie 的改进：

1. 将 n 个元素分成 n / log U 组，每组约 log U 个元素
2. 每组用一棵**平衡 BST**（如红黑树）存储
3. 每组选一个代表元素，所有代表元素组成一棵 **x-fast Trie**

```
原始集合: {1, 3, 5, 7, 9, 11, 13, 15, ...} (假设 w = 8, log U = 8)

分组 (每组 ~8 个):
  组 1: {1, 3, 5, 7, 9, 11, 13, 15} → BST₁, 代表 = 8
  组 2: {17, 19, 21, 23, 25, 27, 29, 31} → BST₂, 代表 = 24
  ...

x-fast Trie 存储: {8, 24, 40, ...}

查询前驱(20):
  1. x-fast Trie 找到 20 的前驱/后继代表 → 代表 24 (组 2)
  2. 在 BST₂ 中找 20 的前驱 → 19
  3. 也可能需要检查相邻组的 BST
  O(log log U) + O(log log U) = O(log log U)
```

| 操作 | 时间 | 说明 |
|------|------|------|
| 查找 | O(log log U) | x-fast + BST |
| 前驱/后继 | O(log log U) | x-fast + BST |
| 插入/删除 | O(log log U) 摊还 | 可能触发组的分裂/合并 |
| 空间 | **O(n)** | 远优于 x-fast |

### 12.4 适用场景

- **理论意义大于实践**：在实际中，由于常数因子和实现复杂度，van Emde Boas 树通常是更实际的选择
- **计算几何**：需要快速前驱/后继的子问题
- **特殊网络路由场景**：值域很大但需要快速最长前缀匹配
- **竞赛/面试**：理解 "对数取对数" 的优化思想

---

## 十三、Concurrent Trie（Ctrie / HAMT）

### 13.1 问题

传统 Trie 在并发场景下需要加锁。全局锁吞吐量低，细粒度锁（per-node）死锁风险高且内存开销大。我们需要一个**无锁（lock-free）**的并发 Trie。

### 13.2 HAMT（Hash Array Mapped Trie）

在讲 Ctrie 之前，先介绍 **HAMT**（Phil Bagwell, 2001）——Ctrie 的基础。

HAMT 是一种将哈希表实现为 Trie 的方式：

1. 对 key 计算 32/64 bit hash
2. 将 hash 按 5 bits 一级分割（每级 32 路分支）
3. 每个内部节点用 **32-bit bitmap** 记录哪些槽位有子节点
4. 子节点紧凑存储在数组中（按 bitmap 中 1 的顺序）

```
hash("cat") = 0b 01101 00011 10010 ...
                 level1 level2 level3

Level 1: bitmap 的第 13 位 (01101) 为 1
         child 数组中的位置 = popcount(bitmap & ((1 << 13) - 1))

Level 2: bitmap 的第 3 位 (00011) 为 1
         ...
```

HAMT 的优势：

- **空间高效**：每个节点只存实际存在的子节点
- **不可变友好**：天然适合持久化数据结构（persistent data structure）

Clojure 的 PersistentHashMap、Scala 的 HashMap、Haskell 的 Data.HashMap 底层都是 HAMT。

### 13.3 Ctrie（Concurrent Trie）

由 Aleksandar Prokopec 在 2012 年提出（博士论文），是 HAMT 的并发版本。

核心机制——三种节点：

```
I-Node（Indirection Node）:
  持有一个指针，指向 C-Node 或 T-Node
  所有 mutation 通过 CAS 更新 I-Node 的指针实现

C-Node（Container Node）:
  内部节点，bitmap + 紧凑子节点数组
  不可变（immutable）——修改时创建新拷贝

S-Node / T-Node（Singleton / Tomb Node）:
  叶节点，存储 key-value 对
  T-Node 标记已删除的节点（用于惰性压缩）
```

树结构：

```
         I-Node (root)
            ↓ CAS
         C-Node [bitmap: 1010]
         /              \
    I-Node              I-Node
       ↓                   ↓ CAS
    S-Node              C-Node [bitmap: 0110]
    (k1,v1)             /              \
                    I-Node              I-Node
                       ↓                   ↓
                    S-Node              S-Node
                    (k2,v2)            (k3,v3)
```

### 13.4 CAS 插入

```
插入 (k4, v4):

1. 计算 hash(k4)
2. 沿着 hash 的每 5 bits 向下遍历
3. 到达目标 I-Node，读取其指向的 C-Node（称为 old_cn）
4. 创建新的 C-Node（称为 new_cn），包含 old_cn 的所有子节点 + 新的 S-Node
5. CAS(I-Node.ptr, old_cn, new_cn)
6. 如果 CAS 成功 → 完成
7. 如果 CAS 失败 → 从根重试（另一个线程抢先修改了）
```

关键点：C-Node 是不可变的（每次修改都创建新的），因此并发读不需要任何同步。只有写操作需要 CAS，且 CAS 的粒度是单个 I-Node——不同子树的修改互不干扰。

### 13.5 O(1) 一致性快照

Ctrie 的一个独特能力：

```scala
def snapshot(): Ctrie = {
    while (true) {
        val old_root = root
        val new_root = new INode(old_root.main)
        if (CAS(root, old_root, new_root)) {
            return new Ctrie(old_root)  // old_root 就是快照
        }
    }
}
```

只需要 CAS 替换根 I-Node。后续的写操作会触发 copy-on-write（遇到旧世代的 I-Node 时先复制再修改），原快照的数据不受影响。

这对于需要一致性迭代（consistent iterator）的场景非常有价值——迭代过程中不受并发写入的影响。

### 13.6 性能特点

| 属性 | 值 |
|------|-----|
| 查找 | O(log₃₂ n) ≈ O(1) 实际（32 路分支，5-6 层就能覆盖数十亿 key） |
| 插入 | O(log₃₂ n) 无锁 |
| 删除 | O(log₃₂ n) 无锁 |
| 快照 | O(1) |
| 迭代 | O(n) 一致性 |

### 13.7 适用场景

- **Scala/JVM 并发编程**：Scala 标准库的 `concurrent.TrieMap` 就是 Ctrie
- **高并发缓存**：读多写少的场景下，无锁读的优势明显
- **需要一致性快照的并发数据结构**：如并发符号表、并发路由表
- **函数式编程语言的并发哈希表**：Ctrie 的不可变 C-Node 天然契合函数式风格

!!! note "Ctrie vs ConcurrentHashMap"
    Java 的 ConcurrentHashMap（CHM）使用分段锁/CAS，查找更快（直接数组索引 vs 多层间接）。但 Ctrie 支持 O(1) 快照和一致性迭代，CHM 不支持。如果不需要快照，CHM 通常是更好的选择。

---

## 十四、其他值得了解的变种

### 14.1 Crit-bit Tree

极简的 Patricia Trie 变种。每个内部节点只存储一个**关键比特位**（critical bit）——即两个子树分歧的那个 bit 位。查找沿着 key 的 bit 位向下，最终在叶节点验证完整 key。

特点：实现极简（约 200 行 C），空间高效，适合嵌入式和高性能场景。D.J. Bernstein 的 `critbit0_tree` 是经典实现。

### 14.2 Qp-Trie（Quelques-bits Popcount Trie）

由 Tony Finch 在 2016 年提出。在 Crit-bit Tree 的基础上，每个内部节点不只看 1 bit，而是看 4-5 bits（一个 nibble），使用 popcount 做紧凑索引。相当于 ART 思想在 bit-level Trie 上的应用。

### 14.3 Judy Array

Doug Baskins 在 HP Labs 开发的高性能 Trie 变种。使用 256 路分支 + 大量的节点类型（17 种！）和压缩技巧。在某些基准测试中比哈希表更快。但实现极其复杂（10 万行 C），难以移植和维护。

### 14.4 HOT（Height Optimized Trie）

2018 年提出的变种，目标是减少 Trie 的高度。HOT 将多层合并为一层（类似 B 树将多层二叉树合并），每个节点处理多个 bit。通过仅存储**区分位（discriminating bits）**而非所有位来压缩。

---

## 十五、变种总览与选型指南

### 15.1 家族关系

```
                        ┌───────────────────────────────┐
                        │          Trie 家族              │
                        └──────────────┬────────────────┘
          ┌────────────────────────────┼──────────────────────────┐
          │                            │                          │
     路径压缩                      空间极致优化                  混合/特殊
  ┌───────┴───────┐            ┌───────┴───────┐          ┌──────┴──────┐
  │               │            │               │          │             │
Radix Tree    Crit-bit      DAWG          Succinct     HAT-Trie    x/y-fast
  │           Qp-Trie     (DAFSA)         Trie         Burst Trie    Trie
  │                                                                    │
  ├── ART (自适应节点)                                                   │
  │     └── HOT (高度优化)                                       整数前驱/后继
  │
  └── Patricia (按 bit 分支)
        └── Crit-bit (极简)

并发:                        数组编码:
  Ctrie (HAMT + CAS)         Double-Array Trie
                              Judy Array
```

### 15.2 选型决策表

| 你的需求 | 推荐变种 | 为什么 |
|----------|----------|--------|
| 简单的前缀匹配 / 自动补全 | 基础 Trie | 实现简单，概念直观 |
| 模糊搜索 / 拼写纠错 | TST | 近似匹配天然高效 |
| HTTP URL 路由 | Radix Tree | 最长前缀匹配，Go httprouter 实战验证 |
| IP 路由查找 | Patricia Trie | 按 bit 分支做最长前缀匹配，网络设备标准方案 |
| 中日韩分词 / 形态分析 | Double-Array Trie | 大字符集下空间紧凑，查找极快，成熟工具链 |
| 内存数据库索引 | ART | 自适应节点 + SIMD + 范围查询 |
| 大规模字符串字典（读多写少） | HAT-Trie | 兼具 Trie 前缀能力和哈希表的 cache 效率 |
| 大规模字符串排序 | Burst Trie | 自适应深度，Burst Sort 性能极佳 |
| 只读词典的极致压缩 | DAWG 或 Succinct Trie | 节点数压缩 10-30 倍 |
| LSM-Tree 的范围过滤器 | Succinct Trie (SuRF) | 替代布隆过滤器，支持范围查询 |
| 高并发键值存储 | Ctrie | 无锁 + O(1) 快照 |
| 整数集前驱/后继 | y-fast Trie | O(log log U) 操作，O(n) 空间 |
| 嵌入式 / 极简实现 | Crit-bit Tree | 200 行 C，极致简洁 |

### 15.3 性能参考（英文词典约 30 万词）

| 变种 | 内存占用 | 查找速度 | 构建速度 | 动态更新 |
|------|----------|----------|----------|----------|
| 基础 Trie (σ=26) | ~150 MB | 快 | 快 | 快 |
| Radix Tree | ~40 MB | 快 | 快 | 快 |
| TST | ~20 MB | 中等 | 快 | 快 |
| Double-Array | ~8 MB | 极快 | 慢 | 困难 |
| ART | ~25 MB | 极快 | 快 | 快 |
| HAT-Trie | ~15 MB | 极快 | 快 | 快 |
| Burst Trie | ~18 MB | 快 | 快 | 中等 |
| DAWG | ~3 MB | 快 | 慢（离线） | 极难 |
| Succinct Trie | ~2 MB | 中等 | 慢（离线） | 不支持 |

> 以上为数量级参考，实际性能取决于数据分布、硬件和实现质量。

---

## 十六、总结

Trie 是一个简洁而强大的思想——**通过共享前缀来组织字符串集合，将查找时间从与集合大小相关变为仅与字符串长度相关**。从这个基础出发，60 多年的研究衍生出了丰富的变种家族：

| 优化方向 | 代表变种 | 核心技术 |
|----------|----------|----------|
| 消除冗余路径 | Radix Tree / Patricia | 路径压缩（合并单子节点链） |
| 自适应节点大小 | ART | 4 种节点类型 + SIMD |
| 共享后缀 | DAWG / DAFSA | DFA 最小化 |
| 数组展平 | Double-Array Trie | base[] + check[] 编码 |
| 极致压缩 | Succinct Trie | LOUDS + rank/select |
| 混合哈希 | HAT-Trie | 上层 Trie + 下层哈希桶 |
| 混合排序容器 | Burst Trie | 上层 Trie + 下层排序数组 |
| 无锁并发 | Ctrie | I-Node + CAS + 不可变 C-Node |
| 整数优化 | x/y-fast Trie | 分层哈希 + 层号二分搜索 |
| 极简实现 | Crit-bit Tree | 每节点只看 1 个关键 bit |

选择哪种变种，本质上是在**查找速度、空间效率、构建成本、更新能力、并发支持、实现复杂度**之间做权衡。理解每种变种背后的设计直觉和工程取舍，比死记它们的复杂度表更有价值。
