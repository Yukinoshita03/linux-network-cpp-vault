# STL 与常用容器入门

## 为什么这页重要

如果目标是写控制面、协议栈周边工具、网络服务或面试代码，`STL` 是必须会用的基础设施。它不是单独几个容器，而是一套“数据结构 + 遍历方式 + 通用算法”的组合。

## STL 的 4 个核心部分

1. 容器：存数据，比如 `vector`、`map`
2. 迭代器：统一遍历容器的方式
3. 算法：比如 `sort`、`find`、`count`
4. 函数对象 / 谓词：给算法传规则，比如自定义排序

最重要的直觉：

- 默认先想 `vector`
- 按 key 查值先想 `unordered_map`
- 需要有序再想 `map` / `set`

## 常用容器速查

### `std::vector`

- 动态数组，最常用
- 支持随机访问
- 尾部插入快
- 适合顺序存储、遍历、排序
- 本身就是一个 RAII 容器：对象活着时管理内部动态数组，析构时自动销毁元素并释放内存

```cpp
vector<int> v = {3, 1, 4};
v.push_back(5);
sort(v.begin(), v.end());
cout << v[0] << "\n";
```

### `std::string`

- 字符串，本质上可以看成字符容器
- 日常工程里会频繁配合 `find`、`substr`、`getline`

### `std::deque`

- 双端队列
- 头尾插入都方便
- 需要两端操作时比 `vector` 更自然

### `std::list`

- 双向链表
- 中间插入删除方便
- 但随机访问差，缓存局部性也差
- 现代 C++ 里很多场景不如 `vector`

### `std::map`

- 有序键值对
- 底层通常是红黑树
- 适合需要按 key 排序、范围查询、稳定有序遍历

```cpp
map<string, int> score;
score["Tom"] = 90;
score["Amy"] = 95;
```

### `std::unordered_map`

- 无序键值对
- 底层是哈希表
- 平均查询更快
- 适合计数、查表、状态索引

```cpp
unordered_map<string, int> cnt;
cnt["apple"]++;
cnt["apple"]++;
```

### `std::set` / `std::unordered_set`

- 都用于去重
- `set` 有序
- `unordered_set` 无序但平均更快

### `std::stack` / `std::queue` / `std::priority_queue`

- 它们是容器适配器
- `stack`：后进先出
- `queue`：先进先出
- `priority_queue`：总能先取到最大值

## 容器选择原则

### 1. 大多数场景先用 `vector`

因为它简单、快、连续存储、算法兼容性最好。

### 2. 需要“名字 -> 值”映射时优先考虑 `unordered_map`

比如：

- 路由表的辅助索引
- 协议邻居状态表
- 单词计数
- 配置项查找

### 3. 需要自动排序时才用 `map` / `set`

比如：

- 想按 key 顺序输出
- 需要区间查询
- 需要 `lower_bound` / `upper_bound`

### 4. 需要频繁取最大值时用 `priority_queue`

比如定时器最早过期事件、调度优先级、Top-K 问题。

## 常用算法

STL 的强项不只是“有容器”，而是“容器 + 算法”一起用。

```cpp
vector<int> v = {5, 2, 8, 2, 1};

sort(v.begin(), v.end());                 // 排序
reverse(v.begin(), v.end());              // 反转
count(v.begin(), v.end(), 2);             // 统计元素个数
find(v.begin(), v.end(), 8);              // 查找元素
max_element(v.begin(), v.end());          // 最大值迭代器
binary_search(v.begin(), v.end(), 5);     // 二分查找，要求已有序
```

## 迭代器的直觉

可以先把迭代器理解成“泛化过的指针”。

- `begin()`：起点
- `end()`：终点后一位
- 很多算法都接收 `[begin, end)` 这一段范围

```cpp
for (auto it = v.begin(); it != v.end(); ++it) {
    cout << *it << " ";
}

for (int x : v) {
    cout << x << " ";
}
```

范围 `for` 更适合日常遍历；需要和算法配合时，再熟悉迭代器写法。

## 常见坑

### `vector` 中间插入和删除不便宜

因为通常要搬动后面的元素。

### `vector` 析构时，`int` 和自定义类型不完全一样

`vector<T>` 逻辑上要负责：

- 销毁当前存活的所有元素对象
- 释放底层内存

但要区分两类 `T`：

1. **像 `int` 这种平凡析构类型**
   - 可以理解成“没有真正需要做的析构工作”
   - 标准库实现通常会把逐个析构优化成几乎什么都不做，直接释放底层内存

2. **自定义类这类非平凡析构类型**
   - `vector` 真的要对每个存活元素调用析构函数
   - 然后再释放底层内存

所以更准确的说法不是：

- “`vector` 对所有类型都一模一样地逐个析构”

而是：

- **从对象语义上，它负责元素销毁；但对 `int` 这类平凡类型，这个销毁步骤通常没有实际成本**

### `map[key]` 可能会隐式插入

如果 key 不存在，`map[key]` 会创建一个默认值。只想查询是否存在时，优先考虑：

```cpp
if (m.find("abc") != m.end()) {
    // found
}
```

### `unordered_map` 无序

不要一边用 `unordered_map`，一边期待它按插入顺序或字典序输出。

### `set` 元素不能直接修改

因为元素本身参与排序，直接改会破坏内部结构。

### 不要默认觉得 `list` 更高级

多数工程代码中，`vector` 更常见，也常常更快。

## 面试和工程里的最低掌握线

至少要熟练：

- `vector`
- `string`
- `map`
- `unordered_map`
- `set`
- `queue`
- `priority_queue`
- `sort`
- `find`
- `count`
- `begin()` / `end()`

## 建议学习顺序

1. `vector` 和 `string`
2. `map` / `unordered_map`
3. `set` / `unordered_set`
4. `queue` / `priority_queue`
5. 迭代器
6. 常用算法

## 一句话记忆

默认先用 `vector`；查表先用 `unordered_map`；有序需求再上 `map` / `set`。

## 相关页面

- [[04-C-Cpp/C 与 C++ 在转发面中的分工]]
- [[01-MOC/学习路线]]
