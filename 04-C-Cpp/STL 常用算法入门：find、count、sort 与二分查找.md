# STL 常用算法入门：find、count、sort 与二分查找

## 先抓住一句话

STL 算法大多不是“某个容器自带的方法”，而是：

- **拿一段迭代器区间 `[first, last)` 来做事**

所以这一页真正想补上的，不是 API 表，
而是这套统一心智：

1. 给算法一段区间
2. 算法按规则扫这段区间
3. 它再把结果用某种形式还给你

常见的返回形式就 3 类：

- 返回迭代器，比如 `find`
- 返回数值，比如 `count`
- 原地修改区间，比如 `sort`、`reverse`

---

## 这一组算法里，最该先会什么

按面试和刷题频率，先掌握这 6 个：

1. `find`
2. `count`
3. `reverse`
4. `sort`
5. `lower_bound` / `upper_bound`
6. `binary_search`

你会发现它们刚好对应三种常见需求：

- 找有没有
- 统计有多少
- 改顺序
- 在有序区间里快速定位

---

## 1. `find`：线性找第一个匹配元素

```cpp
std::vector<int> v = {3, 7, 2, 7, 9};

auto it = std::find(v.begin(), v.end(), 7);
if (it != v.end()) {
    std::cout << *it << "\n";  // 7
}
```

### 它做了什么

它会从 `first` 一直扫到 `last`：

- 遇到第一个等于目标值的元素就停下
- 找不到就返回 `last`

所以它本质上是：

- **顺序扫描**

复杂度通常理解成：

- `O(n)`

### 最关键的返回语义

`find` 返回的不是：

- 下标
- `bool`

而是：

- **迭代器**

这很重要，因为 STL 想保持统一：

- `vector` 可以拿位置
- `list` 没法高效拿下标
- 但所有容器几乎都能返回“指向那个元素的迭代器”

### 最常见的坑

不能直接写：

```cpp
auto it = std::find(v.begin(), v.end(), 100);
std::cout << *it;  // 可能错
```

因为找不到时它会返回 `v.end()`，
而 `end()` 不能解引用。

所以必须先判断：

```cpp
if (it != v.end()) { ... }
```

---

## 2. `count`：线性统计出现次数

```cpp
std::vector<int> v = {3, 7, 2, 7, 9};
auto n = std::count(v.begin(), v.end(), 7);  // n = 2
```

### 它做了什么

也是从头扫到尾。

区别只是：

- `find` 找到第一个就停
- `count` 要把整段都扫完

所以 `count` 返回的是：

- **匹配元素个数**

### 什么时候用 `count`，什么时候用 `find`

如果你想知道：

- “有没有”

那更自然的是：

- `find`

如果你想知道：

- “有几个”

那用：

- `count`

注意：

- `count` 不是更高级的 `find`
- 它通常做的工作更多，因为它不会提早停下

---

## 3. `reverse`：把区间原地翻转

```cpp
std::vector<int> v = {1, 2, 3, 4, 5};
std::reverse(v.begin(), v.end());
// v = {5, 4, 3, 2, 1}
```

### 它做了什么

你可以把它理解成：

- 头尾交换
- 再往中间逼近

也就是一种“双指针相向而行”的感觉。

所以它不是新建一个容器返回，
而是：

- **直接修改原区间**

### 迭代器要求

`reverse` 需要至少双向迭代器，
因为它既要从前往后走，也要从后往前走。

所以：

- `vector` 可以
- `list` 也可以
- 单向迭代器就不够

---

## 4. `sort`：最常用的排序算法入口

```cpp
std::vector<int> v = {5, 1, 4, 2, 3};
std::sort(v.begin(), v.end());
```

### 它做了什么

它会把这段区间按默认 `<` 规则排成升序。

如果你传比较器，也可以自定义规则：

```cpp
std::sort(v.begin(), v.end(), std::greater<int>());
```

### 为什么 `sort` 比前面几个“更挑容器”

因为 `std::sort` 不是只会一步一步扫，
它内部需要频繁做：

- 跳到某个位置
- 算距离
- 快速交换

所以它要求：

- **随机访问迭代器**

这就是为什么：

```cpp
std::sort(v.begin(), v.end());  // vector 可以
```

但：

```cpp
std::list<int> l = {3, 1, 2};
std::sort(l.begin(), l.end());  // 不行
```

`list` 只有双向迭代器，不够强。
如果要给 `list` 排序，要用它自己的成员函数：

```cpp
l.sort();
```

### 你现在应该记住的不是实现细节，而是这句

`sort` 能不能用，
看的不只是容器名，
更关键的是：

- **这个容器给出的迭代器够不够强**

---

## 5. 二分系算法：前提是“区间已经有序”

这一组最容易被误用。

先看：

```cpp
std::vector<int> v = {1, 2, 2, 2, 5, 8};
```

这时：

```cpp
auto it1 = std::lower_bound(v.begin(), v.end(), 2);
auto it2 = std::upper_bound(v.begin(), v.end(), 2);
bool ok = std::binary_search(v.begin(), v.end(), 2);
```

### 它们各自是什么意思

`lower_bound(first, last, x)`：

- 返回这段区间里**第一个不小于 `x`** 的位置
- 也就是第一个 `>= x` 的位置

`upper_bound(first, last, x)`：

- 返回这段区间里**第一个大于 `x`** 的位置
- 也就是第一个 `> x` 的位置

`binary_search(first, last, x)`：

- 只告诉你这段有序区间里**存不存在 `x`**
- 返回 `bool`

### 最重要的前提

这三个算法成立的前提是：

- **区间已经按同一规则排好序**

如果区间没排好，
你再去调 `lower_bound` / `upper_bound` / `binary_search`，
结果就没有你想要的逻辑意义了。

### 一个非常有用的推论

如果区间有序，
某个值出现次数可以写成：

```cpp
auto cnt = std::upper_bound(v.begin(), v.end(), 2)
         - std::lower_bound(v.begin(), v.end(), 2);
```

这在 `vector` 里特别常见，
因为随机访问迭代器支持直接做减法。

---

## 为什么 STL 要把“找”和“二分找”分成两套

因为它们解决的问题环境不一样。

### `find`

- 不要求有序
- 任何能顺序遍历的容器基本都能用
- 代价是线性扫描

### 二分系

- 要求有序
- 换来更快的查找路径

所以这不是“哪个 API 更高级”，
而是：

- **你有没有为更快查找先付出“维护有序”的代价**

这是一个很工程化的 tradeoff。

---

## 在 `map` / `set` 上要注意什么

很多新手会写出这种思路：

```cpp
std::lower_bound(m.begin(), m.end(), x);
```

语义上不一定适合，
性能上也通常不是最优选择。

对于 `map` / `set`，更自然的是用它们自己的成员函数：

```cpp
m.find(key);
m.lower_bound(key);
```

因为：

- `map` / `set` 自己知道底层是树
- 成员函数能直接走树上的搜索路径

而泛型算法版本只看到“迭代器区间”，
它并不知道你的底层是一棵红黑树。

你现阶段先记结论就够：

- `vector` 上常见的是 `sort` + `lower_bound`
- `map` / `set` 上优先想成员函数 `find` / `lower_bound`

---

## 一张速查表

| 算法 | 作用 | 返回什么 | 是否要求有序 |
|---|---|---|---|
| `find` | 找第一个等于目标值的位置 | 迭代器 | 否 |
| `count` | 统计目标值出现次数 | 数量 | 否 |
| `reverse` | 原地翻转区间 | 无 | 否 |
| `sort` | 原地排序 | 无 | 否 |
| `lower_bound` | 找第一个 `>= x` 的位置 | 迭代器 | 是 |
| `upper_bound` | 找第一个 `> x` 的位置 | 迭代器 | 是 |
| `binary_search` | 判断值是否存在 | `bool` | 是 |

---

## 你现在最容易混的 5 件事

1. `find` 返回的是迭代器，不是下标
2. `find` 找不到时返回的是 `end()`，不能直接解引用
3. `count` 会把整段扫完，不会像 `find` 一样提早停
4. `binary_search` 只能回答“有没有”，不能直接给位置
5. `lower_bound` / `upper_bound` 的前提是区间已有序

---

## 面试够用线，至少把这几句话讲顺

1. STL 算法大多吃的是 `[first, last)` 迭代器区间
2. `find` 是线性找，返回迭代器
3. `count` 是线性统计，返回数量
4. `sort` 要求随机访问迭代器，所以 `vector` 能用、`list` 不行
5. `lower_bound` / `upper_bound` / `binary_search` 都要求区间已有序
6. `vector` 上常见套路是排序后再二分

---

## 下一步最自然接什么

学完这页，下一步建议补：

1. `max_element` / `min_element`
2. 自定义比较器
3. `stable_sort`
4. `remove` / `erase` 惯用法
5. `for_each`、`transform`

这样你对算法库的“高频使用层”就比较完整了。

## 相关页面

- [[04-C-Cpp/STL 迭代器入门：本质、区间与算法关系]]
- [[04-C-Cpp/STL 算法进一阶：比较器、max_element 与 remove-erase]]
- [[04-C-Cpp/STL 与常用容器入门]]
- [[04-C-Cpp/C++ 容器与对象语义下一步学习路线]]
