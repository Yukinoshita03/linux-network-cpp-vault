# `unordered_map` 的底层、性能与适用场景

## 为什么这页重要

如果说 `vector` 是“默认容器”，那 `unordered_map` 就是“默认查表容器”。

它特别适合：

- 计数
- 索引
- 状态表
- 名字到对象的映射
- 协议邻居 / 会话 / 连接状态管理

但如果只背“平均 O(1)”，很容易误用。

## 一句话先说本质

`unordered_map` 的底层本质是：

- **哈希表**

它不是按 key 排序存储，而是：

1. 先对 key 做哈希
2. 再根据哈希值决定落到哪个桶（bucket）

所以它的快，主要来自：

- 不需要像树那样一路比较大小找位置
- 而是先算哈希，再去对应桶里找

## 1. 你可以先把它想成什么结构

可以先粗暴理解成：

```text
buckets:
[0] -> 一串元素
[1] -> 一串元素
[2] -> 一串元素
...
```

每个 key 先经过 hash：

```cpp
bucket_index = hash(key) % bucket_count;
```

然后去对应 bucket 里找元素。

所以 `unordered_map` 不是一个“整体有序结构”，而更像：

- 一堆桶
- 每个桶里再放若干元素

## 2. 为什么它平均很快

理想情况下：

- 哈希函数分布均匀
- bucket 足够多
- 每个桶里元素很少

那查找一个 key 时：

1. 先 O(1) 算哈希
2. 再进一个很小的桶里找

所以平均复杂度常写成：

- 插入：平均 O(1)
- 查找：平均 O(1)
- 删除：平均 O(1)

## 3. 为什么它又可能退化

如果哈希分布不好，很多元素都掉进同一个桶：

```text
[0]
[1]
[2] -> 一长串元素
[3]
```

那你查找这个桶里的元素时，就要在桶内挨个比较。

这时复杂度会退化，最坏可接近：

- O(n)

所以你一定要记住：

- `unordered_map` 不是“永远 O(1)”
- 它是“平均 O(1)，最坏可能很差”

## 4. 它和 `map` 的本质差别

### `unordered_map`

- 哈希表
- 无序
- 平均查找更快
- 不支持按 key 有序遍历

### `map`

- 通常是红黑树
- 有序
- 查找、插入、删除通常 O(log n)
- 支持按 key 顺序遍历和范围查询

所以它们最本质的区别不是“哪个高级”，而是：

- `unordered_map` 用哈希换速度
- `map` 用树结构换顺序和稳定的对数复杂度

## 5. 什么时候优先用 `unordered_map`

这些场景最典型：

### 场景 1：计数

```cpp
unordered_map<string, int> cnt;
cnt[word]++;
```

### 场景 2：名字到对象的索引

```cpp
unordered_map<string, Session> sessions;
```

### 场景 3：ID 到状态的查表

```cpp
unordered_map<int, ConnState> states;
```

### 场景 4：协议邻居 / 连接 / 缓存表

你不关心顺序，只关心：

- 快速找到
- 快速更新

这种场景很适合 `unordered_map`。

## 6. 什么时候别急着用它

### 情况 1：你需要有序输出

比如你想按 key 顺序打印：

- 日志
- 配置项
- 排名字典

这时 `map` 更自然。

### 情况 2：你需要范围查询

比如：

- 找所有 key 在某个区间里的元素
- 用 `lower_bound` / `upper_bound`

这时 `unordered_map` 不合适。

### 情况 3：你特别担心最坏情况

树结构的 `map` 更稳定：

- 始终 O(log n)

而 `unordered_map` 更依赖哈希质量和负载情况。

## 7. `unordered_map` 真正快的前提是什么

它快不是白送的，前提通常包括：

1. 哈希函数合理
2. key 可高效哈希
3. 装载因子别太夸张
4. rehash 不要频繁

这就引出两个重要概念：

### `bucket_count`

- 桶的数量

### `load_factor`

- 平均每个桶装多少元素

粗暴理解：

```text
load_factor = size / bucket_count
```

装载因子越大，桶越挤，桶内查找越可能变慢。

## 8. 为什么插着插着会突然变慢一下

因为它可能发生：

- **rehash**

也就是：

1. 分配更多桶
2. 把已有元素重新按新 bucket_count 分布

这和 `vector` 扩容有点像：

- 平时单次操作平均很便宜
- 偶尔某次会因为整体重分布而变贵

所以 `unordered_map` 的平均 O(1) 也是一种均摊意义上的体验。

## 9. 这和 `vector` 有什么相似点

相似点：

- 都有“平时快，偶尔整体重分配”的行为

区别：

- `vector` 重分配是搬连续内存中的元素
- `unordered_map` rehash 是重新分配桶并重新落桶

## 10. 工程上最该形成的直觉

### 默认查表容器

如果你只想：

- key -> value
- 快速查找
- 不关心顺序

先想 `unordered_map`

### 如果你还想要顺序

那就别强拗哈希表，直接想 `map`

### 如果你知道元素很多

可以考虑提前：

- `reserve`

这样能减少 rehash 次数。

## 11. 一句话总结

`unordered_map` 的快，来自哈希分桶而不是有序树查找；它平均 O(1)，但依赖哈希分布和桶负载，不保证最坏情况稳定。

## 12. 这版 MSVC STL 代码该怎么看

如果你在读这版标准库实现，先抓住一个事实：

- `unordered_map` 本体很薄
- 真正逻辑主要在 `xhash`

### 入口 1：`unordered_map` 本体

文件：

- `.../include/unordered_map`

最关键的一行是：

```cpp
class unordered_map : public _Hash<...>
```

这说明：

- `unordered_map` 自己主要负责类型包装
- 底层哈希表能力来自 `_Hash`

### 入口 2：真正底层 `_Hash`

文件：

- `.../include/xhash`

这里有一句非常关键的注释：

```cpp
class _Hash { // hash table -- list with vector of iterators for quick access
```

这告诉你这版实现可以先理解成：

- 元素节点在 `list` 里
- bucket 辅助结构在 `_Vec` 里
- 不是最粗糙的“单独数组 + 每桶裸链表头”心智模型

### 入口 3：插入时是否要 rehash

看这些函数：

- `_Check_rehash_required_1()`
- `_Rehash_for_1()`
- `_Forced_rehash()`

其中 `_Check_rehash_required_1()` 的核心逻辑就是：

- 插入一个新元素后，`new_size / bucket_count()` 会不会超过 `max_load_factor()`

### 入口 4：装载因子和 reserve

看这些函数：

- `load_factor()`
- `max_load_factor()`
- `rehash()`
- `reserve()`

特别是：

```cpp
load_factor = size / bucket_count
reserve(n) -> rehash(足够容纳 n 个元素的桶数)
```

所以 `unordered_map::reserve` 的语义本质是：

- 提前准备足够多的 bucket

不是像 `vector` 那样准备连续元素空间。

### 入口 5：插入路径

在 `unordered_map` 里：

- `operator[]`
- `insert`
- `try_emplace`

很快都会转到基类里的：

- `emplace`
- `_Try_emplace`

所以你读插入流程时，别卡在 `unordered_map` 头文件表面，要往 `xhash` 里的 `_Try_emplace` 和 `emplace` 继续追。

## 13. `_Find_last` 怎么靠 hash 找 bucket

这一版 MSVC `unordered_map` 的心智模型不是“每个 bucket 挂一条独立链表”，而是：

- 元素节点实际放在一个 `list` 里
- bucket 辅助索引放在 `_Vec` 里
- 每个 bucket 在 `_Vec` 里占两个槽位

也就是：

```text
bucket 0 -> _Vec[0] = lo, _Vec[1] = hi
bucket 1 -> _Vec[2] = lo, _Vec[3] = hi
...
```

这里的：

- `lo` 表示这个 bucket 在链表里的第一个元素
- `hi` 表示这个 bucket 在链表里的最后一个元素
- 如果 bucket 为空，两个位置都指向 `end`

这不是讲解时的比喻，而是源码注释字面就在这么说：

```cpp
_Hash_vec<_Aliter> _Vec; // "vector" of list iterators for buckets:
                         // each bucket is 2 iterators denoting the closed range of elements in the bucket,
                         // or both iterators set to _Unchecked_end() if the bucket is empty.
```

也就是说：

- `_Vec` 真的就是 bucket 的边界表
- 每个 bucket 真的占两个槽
- 存的不是元素值，而是指向 `list` 节点的迭代器/指针

所以你看到：

```cpp
_Where     = _Vec[(_Bucket << 1) + 1]
_Bucket_lo = _Vec[_Bucket << 1]
```

不是“玄学写法”，而是在做：

- 先拿 bucket 的右边界 `hi`
- 再拿 bucket 的左边界 `lo`
- 然后只在 `[lo, hi]` 这段链表区间里搜索

插入时这一点也能反证出来。`_Insert_new_node_before()` 会直接维护这两个边界：

```cpp
_Unchecked_iterator& _Bucket_lo = _Bucket_array[_Bucket << 1];
_Unchecked_iterator& _Bucket_hi = _Bucket_array[(_Bucket << 1) + 1];
```

如果 bucket 原来是空的，就把两者都指向新节点；
如果新节点插在最前面，就改 `lo`；
如果新节点插在最后面，就改 `hi`。

所以源码里反复出现：

```cpp
_Bucket << 1
(_Bucket << 1) + 1
```

本质就是：

- 左移一位拿到这个 bucket 的 `lo`
- 再 `+1` 拿到这个 bucket 的 `hi`

## 14. 为什么 bucket 和 `_Mask` 绑定

关键原因是：这版实现强制 `bucket_count` 始终保持为 2 的幂。

你在 `xhash` 的 `_Forced_rehash()` 里会看到：

```cpp
_Buckets = 1 << _Ceiling_of_log_2(_Buckets);
_Mask    = _Buckets - 1;
_Maxidx  = _Buckets;
```

这说明：

- `_Maxidx` 本质上就是当前 bucket 数
- `_Mask = bucket_count - 1`

比如：

- `bucket_count = 8`
- `_Mask = 7`

那么：

```cpp
bucket = hash & _Mask;
```

就等价于：

```cpp
bucket = hash % 8;
```

但这里用的是位运算，不是取模。

所以 bucket 和 `_Mask` 绑定，不是一个零散技巧，而是整套实现的不变量：

1. bucket 数强制拉成 2 的幂
2. 因此可以用 `hash & (bucket_count - 1)` 快速算 bucket 编号
3. rehash 后必须同步更新 `_Mask`

源码内部检查里也明确写了：

```cpp
_Maxidx - 1 == _Mask
```

## 15. `_Find_last` 逐行怎么读

核心代码是：

```cpp
const size_type _Bucket = _Hashval & _Mask;
_Nodeptr _Where         = _Vec._Mypair._Myval2._Myfirst[(_Bucket << 1) + 1]._Ptr;
const _Nodeptr _End     = _List._Mypair._Myval2._Myhead;
if (_Where == _End) {
    return {_End, _Nodeptr{}};
}

const _Nodeptr _Bucket_lo = _Vec._Mypair._Myval2._Myfirst[_Bucket << 1]._Ptr;
for (;;) {
    if (!_Traitsobj(_Keyval, _Traits::_Kfn(_Where->_Myval))) {
        return {_Where->_Next, _Where};
    }

    if (_Where == _Bucket_lo) {
        return {_Where, _Nodeptr{}};
    }

    _Where = _Where->_Prev;
}
```

可以压成 5 步：

### 第 1 步：先用 hash 算出 bucket 编号

```cpp
const size_type _Bucket = _Hashval & _Mask;
```

不是遍历 bucket，也不是做复杂映射，而是直接拿 hash 的低若干位。

### 第 2 步：先取这个 bucket 的尾部 `hi`

```cpp
_Where = _Vec[(_Bucket << 1) + 1]
```

也就是从 bucket 的最后一个元素开始往前看。

这里要特别注意：

- `hi` 是我讲解时用的结构名
- 源码这里没有真的声明一个叫 `_Bucket_hi` 的局部变量

它实际写成了：

```cpp
_Nodeptr _Where = _Vec[(_Bucket << 1) + 1]._Ptr;
```

也就是说：

- `_Vec[(_Bucket << 1) + 1]` 这个位置，语义上就是这个 bucket 的 `hi`
- 只是源码把它直接读进了 `_Where`
- 所以此时的 `_Where`，一开始其实就是“bucket 的 `hi`”

换句话说，这里对应关系是：

```text
讲解名 hi   <=>  源码里的 _Vec[(_Bucket << 1) + 1]
讲解名 lo   <=>  源码里的 _Bucket_lo
```

所以你现在看到的这段源码：

```cpp
_Nodeptr _Where = _Vec[(_Bucket << 1) + 1]._Ptr;
const _Nodeptr _Bucket_lo = _Vec[_Bucket << 1]._Ptr;
```

其实完全可以脑内翻译成：

```cpp
_Nodeptr _Bucket_hi = _Vec[(_Bucket << 1) + 1]._Ptr;
_Nodeptr _Where     = _Bucket_hi;
const _Nodeptr _Bucket_lo = _Vec[_Bucket << 1]._Ptr;
```

只是库作者少起了一个局部变量名，直接把 `hi` 拿来当搜索起点用了。

### 第 3 步：如果 `hi == end`，说明 bucket 空

这时直接返回：

- 插入点是 `end`
- 没有重复元素

也就是说这个 bucket 当前没有任何元素。

### 这 4 行代码当场到底在干什么

```cpp
const size_type _Bucket = _Hashval & _Mask;
_Nodeptr _Where         = _Vec._Mypair._Myval2._Myfirst[(_Bucket << 1) + 1]._Ptr;
const _Nodeptr _End     = _List._Mypair._Myval2._Myhead;
if (_Where == _End) {
    return {_End, _Nodeptr{}};
}
```

可以直接翻译成：

1. 先根据 hash 算出“应该去第几个 bucket”
2. 去 `_Vec` 里拿这个 bucket 的右边界 `hi`
3. 再拿链表的哨兵结点 `end`
4. 如果 `hi == end`，说明这个 bucket 目前是空的
5. 那就直接返回：“没有重复项，插入位置先记成 `end`”

这里几个变量的身份要分清：

- `_Bucket`：bucket 编号
- `_Where`：当前搜索指针，一开始先指向 bucket 的 `hi`
- `_End`：整个链表的哨兵结点，不是某个 bucket 的元素

所以这里的判断：

```cpp
if (_Where == _End)
```

不是在说“找到了结尾所以搜索结束”，而是在说：

- 这个 bucket 的 `hi` 本来就被记录成了 `end`
- 也就是这个 bucket 根本没有元素

于是 `_Find_last` 连扫描都不用扫，直接返回空结果。

### 第 4 步：再取这个 bucket 的头部 `lo`

```cpp
const _Nodeptr _Bucket_lo = _Vec[_Bucket << 1]
```

现在 bucket 在链表里的局部边界就确定了：

- 从 `lo`
- 到 `hi`

所以后面只需要在这段局部范围内搜索，不需要扫整个容器。

### 第 5 步：从 `hi` 往 `lo` 反向扫

这句最关键：

```cpp
if (!_Traitsobj(_Keyval, current_key))
```

在标准情形下，可以粗理解成：

- “`_Keyval < current_key` 不成立了”
- 也就是 `_Keyval >= current_key`

于是：

- 如果当前 key 和 `_Keyval` 等价，那么 `_Where` 会作为 `_Duplicate` 返回
- 插入位置则是 `_Where->_Next`

如果一路退到 `lo` 都还没命中这个条件，说明：

- 新 key 比 bucket 里所有现有元素都更“小”

那就返回：

```cpp
return {_Where, _Nodeptr{}};
```

也就是“插到 bucket 最前面，没有重复项”。

## 16. 为什么叫 `_Find_last`

因为它做的不只是“查重”，还要找插入位置。

它的返回值里有两个信息：

- `_Insert_before`
- `_Duplicate`

对 `unordered_map` 这种唯一 key 容器来说：

- `_Duplicate` 非空，说明已经有同 key 元素，不能插
- `_Duplicate` 为空，说明可以在 `_Insert_before` 前插入

你可以把它理解成：

- 先定位 bucket
- 再在 bucket 的局部区间里，找“最后一个合适位置”之后的插入点

## 17. 一句话压缩这段机制

这版 MSVC `unordered_map` 的 bucket 定位机制是：

- `rehash` 时把 bucket 数强制拉成 2 的幂
- 令 `_Mask = bucket_count - 1`
- 查找时用 `hash & _Mask` 直接算 bucket 编号
- 再通过 `_Vec` 里的 `lo/hi` 边界，只在该 bucket 对应的链表局部区间里搜索

所以 `_Find_last` 的本质不是“全表查找”，而是：

- 先位运算定位 bucket
- 再局部扫描 bucket 对应的那一段链表

## 18. 为什么 `_Hashval` 已经是 hash 了，还要再 `& _Mask`

对，这里的 `_Hashval` 确实已经是：

- `hash(key)` 算出来的结果

但要注意：

- **hash 值** 不是 **bucket 编号**

这是两个层次的东西。

你可以把流程拆成：

```text
key
-> hash(key)
-> 得到一个很大的整数 _Hashval
-> 再把这个整数映射到 [0, bucket_count-1]
-> 得到 bucket 编号
```

所以：

```cpp
_Hashval = hash(key)
_Bucket  = _Hashval & _Mask
```

不是“重复 hash”，而是：

- 第一句是在算 hash
- 第二句是在把 hash 值映射到当前 bucket 范围里

## 19. 为什么不能直接把 `_Hashval` 当 bucket 用

因为 hash 值通常很大，但 bucket 数很小。

比如：

- `bucket_count = 8`
- `_Hashval = 123456`

那 bucket 只能是：

- `0` 到 `7`

不可能直接拿 `123456` 去当 bucket 下标。

所以必须做一次“压缩映射”：

```cpp
bucket = hash % bucket_count
```

而这版实现因为 `bucket_count` 总是 2 的幂，所以把它优化成：

```cpp
bucket = hash & (bucket_count - 1)
```

也就是：

```cpp
_Bucket = _Hashval & _Mask
```

## 20. 你可以把它理解成“先算身份证，再分柜子”

一个很直观的类比是：

- `hash(key)` 像是给这个 key 算出一个“指纹号”
- `bucket` 像是把这个指纹号分配到某一个柜子

所以：

- `_Hashval` 决定“它大概属于哪一类”
- `_Bucket` 决定“它现在去哪个桶里找”

前者是全空间的 hash 编码。
后者是当前这张哈希表、在当前 bucket 数下的落桶结果。

## 21. 这版 MSVC `unordered_map` 的整体结构图

如果你现在觉得一会儿 `list`，一会儿 `_Vec`，很乱，那先记一句：

- **元素本体在 `list` 里**
- **bucket 索引在 `_Vec` 里**

也就是说，这版实现不是：

- 一个数组
- 每个数组槽挂一条独立链表

而是：

- 全部元素先串在一条总链表 `_List` 里
- `_Vec` 只负责记“每个 bucket 在这条总链表中的哪一段”

### 先看最粗图

```text
unordered_map
|
|-- _List   : 真正存元素的双向链表
|
|-- _Vec    : bucket 边界表
|            每个 bucket 占 2 个槽
|            [lo, hi]
|
|-- _Mask   : 用来把 hash 映射成 bucket 编号
|
`-- _Maxidx : bucket_count
```

### 再看一个具体例子

假设现在有 4 个 bucket：

```text
bucket_count = 4
_Mask = 3
```

链表里当前元素长这样：

```text
_List:

[head]
  |
  v
 [k1] <-> [k9] <-> [k2] <-> [k6] <-> [k10] <-> [k3]
```

注意：

- 这不是按 key 全局有序
- 也不是所有元素乱放
- 而是“同一个 bucket 的元素会在链表里形成一段局部区间”

比如可以想成：

```text
bucket 0 : [k1] [k9]
bucket 1 : [k2]
bucket 2 : [k6] [k10]
bucket 3 : [k3]
```

于是 `_Vec` 里记的不是值，而是这些区间的左右边界：

```text
_Vec:

bucket 0 -> _Vec[0] = lo -> [k1]
            _Vec[1] = hi -> [k9]

bucket 1 -> _Vec[2] = lo -> [k2]
            _Vec[3] = hi -> [k2]

bucket 2 -> _Vec[4] = lo -> [k6]
            _Vec[5] = hi -> [k10]

bucket 3 -> _Vec[6] = lo -> [k3]
            _Vec[7] = hi -> [k3]
```

如果某个 bucket 为空，就会这样：

```text
bucket i -> lo = end
            hi = end
```

### 为什么要有两个结构

因为它们职责不同。

`_List` 负责：

- 真正存节点
- 提供稳定节点位置
- 支持在已知位置前后插入删除

`_Vec` 负责：

- 从 bucket 编号快速找到这个 bucket 对应的区间
- 避免每次查找都扫整条链表

所以 `_Vec` 不是“第二份元素存储”，而是：

- **对 `_List` 的索引**

### 查找时怎么走

查找某个 key，大致流程就是：

```text
key
-> hash(key)
-> _Hashval
-> _Bucket = _Hashval & _Mask
-> 去 _Vec 里拿这个 bucket 的 [lo, hi]
-> 只在 _List 的这段区间里搜
```

图上看就是：

```text
key
  |
  v
hash(key)
  |
  v
bucket = hash & _Mask
  |
  v
_Vec[2*bucket], _Vec[2*bucket+1]
  |
  v
_List 上的局部区间 [lo ... hi]
```

### 插入时怎么走

插入也不是直接插进 `_Vec`。

真正动作是：

1. 先根据 hash 找 bucket
2. 在 bucket 对应区间里找重复项或插入点
3. 把新节点插进 `_List`
4. 再更新 `_Vec` 里这个 bucket 的 `lo/hi`

所以 `_Vec` 只是“记边界”，真正节点还是在 `_List` 里。

### 一句话收拢

你可以把这版实现理解成：

- `_List` 是仓库里真正摆货的货架
- `_Vec` 是每个 bucket 对应货架区间的目录卡

查找时不是满仓乱找，而是：

- 先算出该看哪个目录卡
- 再只去那一小段货架里找

这就是为什么源码里你会同时看到：

- `_List`
- `_Vec`
- `_Bucket_lo`
- `_Bucket_hi`

它们不是重复，而是在配合完成“**链表存储 + bucket 边界索引**”这套结构。

## 22. bucket 的这两个“index”到底怎么存

这里最容易误会的一点是：

- 它存的不是两个整数下标
- 也不是两个数组位置编号

而是：

- **两个迭代器/指针槽位**

源码里 `_Vec` 的注释已经说了：

```cpp
_Hash_vec<_Aliter> _Vec; // "vector" of list iterators for buckets:
                         // each bucket is 2 iterators ...
```

所以每个 bucket 真正存的是：

- `lo`：指向这个 bucket 在 `_List` 里的第一个节点
- `hi`：指向这个 bucket 在 `_List` 里的最后一个节点

### 内存排布长什么样

假设有 4 个 bucket，那么 `_Vec` 会分配：

```text
_Buckets << 1 = 8
```

个槽位。

也就是：

```text
_Vec[0] = bucket 0 的 lo
_Vec[1] = bucket 0 的 hi

_Vec[2] = bucket 1 的 lo
_Vec[3] = bucket 1 的 hi

_Vec[4] = bucket 2 的 lo
_Vec[5] = bucket 2 的 hi

_Vec[6] = bucket 3 的 lo
_Vec[7] = bucket 3 的 hi
```

注意这里每个槽里放的不是 `0/1/2/3` 这种整数，而是类似：

```text
pointer/iterator -> 某个 list 节点
```

如果 bucket 为空，就放：

```text
end iterator
```

### 为什么你会觉得像“index”

因为访问方式长这样：

```cpp
_Vec[_Bucket << 1]
_Vec[(_Bucket << 1) + 1]
```

这看起来像“先算下标再取值”，所以很像在存 index。

但实际上：

- 这里 `[_Bucket << 1]` 和 `[(_Bucket << 1)+1]` 是 **访问 `_Vec` 这个数组的槽位下标**
- 槽位里面存的内容，却是 **list 迭代器/节点指针**

所以有两层：

1. `_Vec` 自己是一个线性数组，需要用整数下标访问
2. `_Vec` 的每个元素，存的是“指向 `_List` 节点的位置”

### 可以把它脑内翻译成这个结构

```text
bucket_table = [
  { lo: ptr_to_node, hi: ptr_to_node },
  { lo: ptr_to_node, hi: ptr_to_node },
  { lo: ptr_to_node, hi: ptr_to_node },
  ...
]
```

只不过标准库为了实现简单和内存布局直接，没真写成结构体数组，而是压平成了：

```text
[lo0, hi0, lo1, hi1, lo2, hi2, ...]
```

### 一句话钉死

这版 MSVC `unordered_map` 里，bucket 的两个“index”本质上不是数字索引，而是：

- `bucket -> _Vec` 中的两个槽位
- 槽位里再存“指向 `_List` 节点的迭代器/指针”

所以它是：

- **数组定位 bucket**
- **指针定位 bucket 在链表里的边界**

## 23. `_Insert_new_node_before()` 插入后怎么维护 `lo/hi`

这个函数的关键不是“找位置”，而是：

- 节点已经知道要插到哪里了
- 现在要把它接进 `_List`
- 然后把这个 bucket 的边界修正好

核心代码是：

```cpp
const size_type _Bucket         = _Hashval & _Mask;
_Unchecked_iterator& _Bucket_lo = _Bucket_array[_Bucket << 1];
_Unchecked_iterator& _Bucket_hi = _Bucket_array[(_Bucket << 1) + 1];
if (_Bucket_lo._Ptr == _Head) {
    _Bucket_lo._Ptr = _Newnode;
    _Bucket_hi._Ptr = _Newnode;
} else if (_Bucket_lo._Ptr == _Insert_before) {
    _Bucket_lo._Ptr = _Newnode;
} else if (_Bucket_hi._Ptr == _Insert_after) {
    _Bucket_hi._Ptr = _Newnode;
}
```

先按逻辑翻译：

1. 先把 `_Newnode` 真正接到双向链表 `_List` 里
2. 再找到它所属 bucket 的 `lo/hi`
3. 根据新节点插入的位置，决定要不要改边界

只有 3 种情况：

### 情况 1：bucket 原来是空的

```cpp
if (_Bucket_lo._Ptr == _Head)
```

说明：

- 原来 `lo == hi == end`
- 现在这个 bucket 第一次有元素

所以：

- `lo = newnode`
- `hi = newnode`

### 情况 2：新节点插到了 bucket 最前面

```cpp
else if (_Bucket_lo._Ptr == _Insert_before)
```

意思是：

- 原来的 `lo` 就是这次插入位置前面的那个节点
- 那新节点一定比原 `lo` 更靠前

所以只需要：

- `lo = newnode`

`hi` 不变。

### 情况 3：新节点插到了 bucket 最后面

```cpp
else if (_Bucket_hi._Ptr == _Insert_after)
```

意思是：

- 插入位置前面的节点正好是原来的 `hi`
- 那新节点一定落在 bucket 的最右边

所以只需要：

- `hi = newnode`

`lo` 不变。

### 如果这 3 种都不是

那就说明：

- 新节点插在 bucket 中间

这时：

- `lo` 不用改
- `hi` 也不用改

因为 bucket 的左右边界并没有变化。

### 一句话收拢

`_Insert_new_node_before()` 的边界维护思想非常朴素：

- 插前面，改 `lo`
- 插后面，改 `hi`
- 插中间，不改
- 原来为空，就同时改 `lo/hi`

## 24. 为什么 `rehash` 只重建 `_Vec`，节点还在 `_List`

这个点本质上是在问：

- rehash 到底是在“搬元素”
- 还是在“重建索引”

对这版实现来说，更接近第二种。

看 `_Forced_rehash()`：

```cpp
_Vec._Assign_grow(_Buckets << 1, _End);
_Mask   = _Buckets - 1;
_Maxidx = _Buckets;
```

这一步先做的是：

- 把新的 bucket 边界表 `_Vec` 重新分配出来
- 全部初始化成 `end`

注意这里没新建元素节点，节点本体还都在 `_List` 里。

然后它开始遍历 `_List`：

```cpp
for (_Unchecked_iterator _Next_inserted = _Inserted; _Inserted != _End; _Inserted = _Next_inserted) {
    ++_Next_inserted;
    const size_type _Bucket = bucket(_Inserted_key);
    ...
}
```

它做的事情是：

1. 重新按新的 `_Mask` / bucket 数，给每个旧节点算新 bucket
2. 更新新 `_Vec` 中该 bucket 的 `lo/hi`
3. 如果需要，为了让同 bucket 元素继续形成局部区间，会在 `_List` 里做 `splice`

这里的 `splice` 很重要：

- 它不是“重新构造新节点”
- 也不是“拷贝 value”
- 而是把**已有链表节点改链**

所以本质上：

- 节点对象没被销毁重建
- 只是链表顺序和 bucket 索引被重新整理了

### 为什么能这么做

因为这版实现从一开始就是：

- `_List` 存储节点本体
- `_Vec` 只是 bucket 边界索引

既然 bucket 信息只是“指向 `_List` 某一段的边界”，那 bucket 数变了时，最核心要重建的当然是：

- `_Vec`

而不是 value 节点本身。

### 一句话收拢

这版 `rehash` 的核心不是“把元素重新申请一遍”，而是：

- **节点留在 `_List`**
- **bucket 视图 `_Vec` 全部重建**
- **必要时通过 `splice` 调整链表顺序，让每个 bucket 仍对应一段局部区间**

## 25. 再切到 `map/xtree`，底层思路为什么完全不同

`unordered_map` 的查找路径是：

```text
key
-> hash(key)
-> bucket
-> bucket 对应局部区间
```

它依赖的是：

- 哈希函数
- bucket 分布
- 平均 O(1)

而 `map` 的 `xtree` 根本不是这条路。

它是红黑树模型。

看 `xtree` 的 `_Find_lower_bound()`：

```cpp
while (!_Trynode->_Isnil) {
    if (_DEBUG_LT_PRED(_Getcomp(), _Traits::_Kfn(_Trynode->_Myval), _Keyval)) {
        _Trynode = _Trynode->_Right;
    } else {
        _Result._Bound = _Trynode;
        _Trynode       = _Trynode->_Left;
    }
}
```

这说明 `map` 的查找本质是：

- 从根开始
- 不断做 key 比较
- 左走 / 右走
- 最后找到插入位置或下界

所以它依赖的是：

- 比较器
- 树平衡性质
- 稳定的 O(log n)

### 两者最本质的区别

`unordered_map`：

- 先 hash，再落桶
- 关注“分到哪个 bucket”
- 结构核心是“节点存储 + bucket 索引”

`map`：

- 直接比较 key 大小
- 关注“在树上往左还是往右”
- 结构核心是“父子指针 + 平衡维护”

### 插入时的差异也很明显

`unordered_map` 插入：

- 先找 bucket
- 再找 bucket 内位置
- 接进 `_List`
- 修 `lo/hi`

`map` 插入：

- 先树搜索找插入位置
- 接成某个父节点的左/右孩子
- 再做红黑树旋转/染色修复平衡

### 一句话钉死

你可以把它们区分成两种完全不同的“组织信息”的方式：

- `unordered_map`：**按 hash 分桶**
- `map`：**按 key 大小排成平衡树**

所以前者读源码时你会反复看到：

- `_Mask`
- `_Vec`
- bucket
- rehash

后者读源码时你会反复看到：

- `_Left`
- `_Right`
- `_Parent`
- lower_bound
- 插入修复 / 平衡维护

## 相关页面

- [[04-C-Cpp/STL 与常用容器入门]]
- [[04-C-Cpp/vector 扩容、异常安全与 noexcept]]
- [[04-C-Cpp/C++ 容器与对象语义下一步学习路线]]
