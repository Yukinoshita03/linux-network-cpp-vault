# STL 算法进一阶：比较器、max_element 与 remove-erase

## 这页是干嘛的

如果你已经会了：

- `find`
- `count`
- `reverse`
- `sort`
- `lower_bound` / `upper_bound` / `binary_search`

那算法库下一步最值钱的，不是继续背一堆 API，
而是把下面这 4 个点补齐：

1. `max_element` / `min_element`
2. 自定义比较器
3. `stable_sort`
4. `remove` / `erase` 惯用法

这几块一补上，算法库就更接近“能打题、能面试、能写工程代码”的状态了。

---

## 1. `max_element` / `min_element`

```cpp
std::vector<int> v = {5, 2, 9, 1, 7};

auto it1 = std::max_element(v.begin(), v.end());
auto it2 = std::min_element(v.begin(), v.end());
```

### 它们返回什么

返回的是：

- **迭代器**

不是数值本身。

所以通常你会写：

```cpp
if (it1 != v.end()) {
    std::cout << *it1 << "\n";
}
```

### 它们怎么做

本质就是：

- 从头扫到尾
- 维护当前最大值 / 最小值

所以复杂度通常是：

- `O(n)`

### 你要记住的直觉

`max_element` 和 `find` 很像：

- 都是线性扫
- 都返回迭代器

只不过：

- `find` 的判定条件是“等不等于目标值”
- `max_element` 的判定条件是“谁更大”

---

## 2. 比较器：你在告诉算法“什么叫更小”

比如默认 `sort` 用的是 `<`：

```cpp
std::sort(v.begin(), v.end());
```

如果你想降序：

```cpp
std::sort(v.begin(), v.end(), std::greater<int>());
```

或者自己写比较规则：

```cpp
std::sort(v.begin(), v.end(), [](int a, int b) {
    return a > b;
});
```

### 比较器本质是什么

比较器本质上是在告诉算法：

- **谁应该排在前面**

也就是告诉它：

- “什么算更小”

很多人误以为比较器是在问：

- “要不要交换”

但更稳定的理解应该是：

- 如果 `comp(a, b)` 为真，就表示 `a` 应该排在 `b` 前面

---

## 3. `stable_sort`：相等元素保持原相对顺序

```cpp
std::stable_sort(v.begin(), v.end(), comp);
```

### 它和 `sort` 的区别

普通 `sort` 只保证最后排好序，
但：

- 相等元素之间原本的先后顺序，可能被打乱

`stable_sort` 则额外保证：

- 如果两个元素按比较规则看“相等”
- 那它们原来的前后关系会被保留

### 什么时候这个性质重要

比如你有一组学生：

- 先按姓名排过一次
- 再按分数排

如果第二次你想让“分数相同的人继续保持姓名顺序”，
那 `stable_sort` 就有意义。

### 你现阶段先记结论

- 默认先想到 `sort`
- 只有当“相等元素保序”这件事重要时，再想到 `stable_sort`

---

## 4. `remove` 最容易坑人的地方：它不真的删元素

先看这个经典写法：

```cpp
std::vector<int> v = {1, 2, 3, 2, 4};
auto it = std::remove(v.begin(), v.end(), 2);
```

很多人以为这一步之后：

- 容器里所有 `2` 都没了

其实不是。

### `remove` 真正在做什么

它做的是：

- 把“要保留的元素”往前搬
- 返回一个“新的逻辑结尾”

也就是说：

- 前半段是你想保留的数据
- 后半段变成一坨“可以忽略的旧值垃圾区”

所以 `remove` 本质不是 delete，
而是：

- **重排区间**

### 为什么还要接 `erase`

因为对 `vector` 这种容器来说，
真正改 `size()`、把尾巴裁掉的是：

```cpp
v.erase(it, v.end());
```

所以正确惯用法是：

```cpp
v.erase(std::remove(v.begin(), v.end(), 2), v.end());
```

这就叫：

- **remove-erase idiom**

---

## 为什么 `remove` 会长成这种怪样子

因为 STL 算法和容器是分离的。

`remove` 只是一个泛型算法，
它只拿到：

- 一段迭代器区间

它并不知道：

- 你背后是不是 `vector`
- 能不能改容器大小
- 有没有 `erase`

所以它只能做自己有能力做的事：

- 重排元素
- 返回新区间结尾

至于“真正缩容”，那是容器成员函数去做。

这个设计非常 STL。

---

## `remove_if`：把“删谁”变成一个规则

```cpp
v.erase(std::remove_if(v.begin(), v.end(), [](int x) {
    return x % 2 == 0;
}), v.end());
```

这表示：

- 把所有偶数删掉

你会发现这里已经自然和 `lambda` 连起来了。

所以算法库补到这一步之后，
下一块最自然接的就是：

- **lambda**

---

## 这一页真正该记住的 5 句话

1. `max_element` / `min_element` 返回的是迭代器，不是数值
2. 比较器本质是在告诉算法“谁应该排前面”
3. `stable_sort` 比 `sort` 多出来的是“相等元素保序”
4. `remove` 不会真的删容器元素，它只是重排并返回新区间结尾
5. 真正把尾巴裁掉，要配合 `erase`

---

## 面试够用线

至少要能顺着讲清：

1. `max_element` 是线性扫描找最大值，返回迭代器
2. `sort` 可以传比较器来自定义排序规则
3. `stable_sort` 和 `sort` 的区别是相等元素是否保序
4. `remove` 不改容器大小，所以 `vector` 上删除元素常写成 `erase(remove(...), end())`
5. `remove_if` 往往和 `lambda` 一起出现

---

## 下一步最自然接什么

如果这一页也会了，
那我建议别在 STL 算法库里继续铺太多了，
下一步直接切到：

1. `lambda`
2. 智能指针：`unique_ptr` / `shared_ptr` / `weak_ptr`

因为这两块比继续抠更多算法 API 更值钱。

## 相关页面

- [[04-C-Cpp/STL 常用算法入门：find、count、sort 与二分查找]]
- [[04-C-Cpp/C++ lambda 入门：本质、捕获与算法配合]]
- [[04-C-Cpp/STL 迭代器入门：本质、区间与算法关系]]
- [[04-C-Cpp/C++ 容器与对象语义下一步学习路线]]
