 # `map` 与红黑树底层实现

## 为什么要单独学 `map`

如果说 `unordered_map` 的核心问题是：

- hash 怎么落 bucket
- bucket 怎么索引节点

那 `map` 的核心问题完全不是这个。

`map` 的问题是：

- 一个 key 应该插到树的哪里
- 怎么保证树别退化成链表
- 为什么它能稳定做到 `O(log n)`

所以你要把 `map` 和 `unordered_map` 明确分开看：

- `unordered_map` 是哈希表
- `map` 是有序平衡树

## 1. 一句话先说本质

`map` 的本质是：

- **按 key 有序存储的红黑树**

你在 MSVC STL 的 [map](</C:/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.43.34808/include/map:71>) 里直接能看到：

```cpp
class map : public _Tree<...> {
    // ordered red-black tree of {key, mapped} values, unique keys
```

这句话已经把底层说死了：

- ordered
- red-black tree
- unique keys

所以 `map` 不是自己实现了一套复杂逻辑，它本体很薄，真正底层都在：

- `xtree`

## 2. `map` 本体很薄，真正逻辑在 `xtree`

`map` 头文件里最关键的是这行：

```cpp
class map : public _Tree<_Tmap_traits<...>>
```

这说明：

- `map` 主要负责把 `key/value/compare/allocator` 这些类型拼起来
- 真正的树结构、查找、插入、删除、迭代器都在 `_Tree`

你看 `operator[]`、`insert`、`try_emplace` 这些入口，最后都会落到 `_Tree` 的能力上。

比如 [map](</C:/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.43.34808/include/map:196>) 的 `_Try_emplace()`：

```cpp
const auto _Loc = _Mybase::_Find_lower_bound(_Keyval);
...
return {_Scary->_Insert_node(_Loc._Location, _Inserted), true};
```

它其实就是：

1. 先树搜索找位置
2. 再把新节点插进去

## 3. 红黑树节点到底长什么样

在 [xtree](</C:/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.43.34808/include/xtree:321>) 里，节点结构是：

```cpp
struct _Tree_node {
    _Nodeptr _Left;
    _Nodeptr _Parent;
    _Nodeptr _Right;
    char _Color;
    char _Isnil;
    value_type _Myval;
};
```

这就是一个标准的树节点模型：

- `_Left`：左孩子
- `_Right`：右孩子
- `_Parent`：父节点
- `_Color`：红 or 黑
- `_Myval`：真正存的值，也就是 `pair<const Key, T>`

所以 `map<int, string>` 的每个节点里，本质上是：

```text
{ left*, parent*, right*, color, key-value }
```

## 4. 它还有一个很关键的“头结点 / 哨兵结点”

这版实现不是用 `nullptr` 到处判空，而是有一个特殊的 head 节点。

你看 [xtree](</C:/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.43.34808/include/xtree:343>) 的 `_Buyheadnode()`：

```cpp
_Pnode->_Left   = _Pnode;
_Pnode->_Parent = _Pnode;
_Pnode->_Right  = _Pnode;
_Pnode->_Color  = _Black;
_Pnode->_Isnil  = true;
```

也就是说这个 head 节点一开始：

- 左右父都指向自己
- 是黑色
- `_Isnil = true`

这个 head 很重要，它一口气承担了几个角色：

- `end()` 对应的哨兵
- 空树时的统一空节点
- `head->_Parent` 记整棵树的 root
- `head->_Left` 记最小节点
- `head->_Right` 记最大节点

所以你可以脑补成：

```text
head
├─ Parent -> root
├─ Left   -> 最小节点
└─ Right  -> 最大节点
```

这就是为什么树迭代器能很自然地做：

- `begin()` 找最小节点
- `end()` 指向 head

## 5. `map` 为什么天然有序

因为树上的位置就是由 key 比较决定的。

规则非常简单：

- 比当前节点小，往左走
- 比当前节点大，往右走

所以整棵树满足：

- 左子树 key 都更小
- 右子树 key 都更大

这就直接带来了：

- 中序遍历结果天然有序

所以 `map` 不需要像 `unordered_map` 那样额外想“怎么输出有序”，它结构本身就保证了顺序。

## 6. `_Find_lower_bound()` 是 `map` 的灵魂路径

如果你只选一个函数读 `map`，最值得读的是：

- `_Find_lower_bound()`

看 [xtree](</C:/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.43.34808/include/xtree:1618>)：

```cpp
while (!_Trynode->_Isnil) {
    _Result._Location._Parent = _Trynode;
    if (_DEBUG_LT_PRED(_Getcomp(), _Traits::_Kfn(_Trynode->_Myval), _Keyval)) {
        _Result._Location._Child = _Tree_child::_Right;
        _Trynode                 = _Trynode->_Right;
    } else {
        _Result._Location._Child = _Tree_child::_Left;
        _Result._Bound           = _Trynode;
        _Trynode                 = _Trynode->_Left;
    }
}
```

先别怕，逻辑非常清楚。

### 它在做什么

它在找：

- “第一个不小于 `_Keyval` 的节点”

也就是 lower_bound 的标准定义。

### 怎么找

假设当前在节点 `cur`：

#### 情况 1：`cur.key < key`

说明：

- `cur` 太小了
- lower_bound 不可能在 `cur` 或它左边

所以：

- 往右走

#### 情况 2：`cur.key >= key`

说明：

- `cur` 有资格成为 lower_bound 候选

于是：

- 先把它记到 `_Result._Bound`
- 再继续往左走，看能不能找到一个更小但仍然 `>= key` 的节点

### 最后得到什么

最后它返回两个核心信息：

- `_Bound`：找到的 lower_bound 候选节点
- `_Location`：如果要插入，新节点应该挂到哪个父节点的左边或右边

这就是为什么这个函数既服务查找，也服务插入。

## 7. 插入流程怎么走

插入的主线在 [xtree](</C:/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.43.34808/include/xtree:1000>) 的 `_Emplace()`。

逻辑可以压成：

1. 先提取 key
2. 调 `_Find_lower_bound(key)` 找位置
3. 如果是唯一键容器，检查是否重复
4. 创建新节点
5. 调 `_Insert_node(...)` 接到树上
6. 做红黑树修复，恢复平衡

也就是说，`map` 插入不是：

- 找 bucket
- 接链表

而是：

- 找父节点
- 挂左孩子或右孩子
- 然后平衡树

## 8. 为什么它能稳定 `O(log n)`

因为它不是普通二叉搜索树，而是：

- 红黑树

红黑树的目标不是“绝对最矮”，而是：

- 别太歪

它通过颜色约束保证：

- 从根到叶子的最长路径不会夸张地长

所以：

- 查找 `O(log n)`
- 插入 `O(log n)`
- 删除 `O(log n)`

比起哈希表的“平均快，最坏可能退化”，`map` 的优势是：

- **稳定**

## 9. 旋转到底是什么

树想保持平衡，不能只是硬插，必要时要“转一下”。

这就是左旋和右旋。

在 [xtree](</C:/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.43.34808/include/xtree:473>) 里有：

- `_Lrotate`
- `_Rrotate`

你可以先把旋转理解成：

- 不改变中序顺序
- 但改变局部父子关系
- 让树别长歪

### 左旋直觉图

```text
    A                  B
     \                / \
      B      ->      A   C
       \
        C
```

### 右旋直觉图

```text
        C              B
       /              / \
      B      ->      A   C
     /
    A
```

旋转不是为了排序，而是为了：

- 维护平衡

## 10. `map` 和 `unordered_map` 最本质的区别

### `unordered_map`

路径是：

```text
key -> hash -> bucket -> bucket 内搜索
```

核心词是：

- hash
- bucket
- load_factor
- rehash

### `map`

路径是：

```text
key -> 比较大小 -> 树上左走/右走 -> 位置
```

核心词是：

- left/right/parent
- lower_bound
- rotate
- red/black

所以两者不是“一个快一个慢”这么简单，而是两种完全不同的组织信息方式：

- `unordered_map`：按 hash 分桶
- `map`：按 key 大小排成平衡树

## 11. 你读 `map` 源码时最该盯的 5 个点

如果你接下来继续读源码，优先盯这几个：

1. `_Tree_node`
   先把节点模型吃透。

2. `_Buyheadnode`
   先理解 head/sentinel 在这版实现里的角色。

3. `_Find_lower_bound`
   这是查找和插入共用的核心搜索路径。

4. `_Emplace`
   看插入整体流程怎么串起来。

5. `_Lrotate` / `_Rrotate`
   建立“平衡维护”的第一直觉。

## 12. 一句话总结

`map` 的底层不是哈希表，而是一棵按 key 有序组织的红黑树：节点里保存左右孩子、父节点和颜色；查找靠大小比较一路左走右走；插入先用 `lower_bound` 找位置，再挂节点并通过旋转和染色维持树的平衡，因此它稳定提供 `O(log n)` 的查找、插入和删除。

## 相关页面

- [[04-C-Cpp/STL 与常用容器入门]]
- [[04-C-Cpp/unordered_map 的底层、性能与适用场景]]
- [[04-C-Cpp/C++ 容器与对象语义下一步学习路线]]
