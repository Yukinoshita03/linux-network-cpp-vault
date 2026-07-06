# C++ std function 入门：lambda、回调与统一可调用类型

## 先给一句话定义

`std::function` 可以先理解成：

- **一个能装“可调用对象”的通用盒子**

只要某个东西“能像函数一样被调用”，并且调用签名对得上，
它就有机会装进 `std::function`。

所以它最核心的价值不是“执行函数”，
而是：

- **把不同种类的可调用对象，统一成一个明确的类型**

---

## 为什么学完 lambda，下一步自然就是 `std::function`

你前面已经学到：

- 普通函数
- lambda
- 带捕获的 lambda

问题是：

- lambda 的类型其实是匿名的
- 每个 lambda 基本上都是一个不同的类型

所以如果你想做这种事：

- 把一个回调存到变量里
- 把回调作为成员保存起来
- 把多个回调放进容器
- 写一个统一接口，接受“任何符合签名的可调用对象”

那就需要一个统一外壳。

这个外壳就是：

- `std::function`

---

## 先看最小例子

```cpp
#include <functional>
#include <iostream>

int add(int a, int b) {
    return a + b;
}

int main() {
    std::function<int(int, int)> f = add;
    std::cout << f(3, 4) << "\n"; // 7
}
```

这里：

```cpp
std::function<int(int, int)>
```

表示：

- 这是一个可调用对象
- 接收两个 `int`
- 返回一个 `int`

你可以把它读成：

- “函数签名长成 `int(int, int)` 的通用包装器”

---

## 它能装什么

### 1. 普通函数

```cpp
int add(int a, int b) {
    return a + b;
}

std::function<int(int, int)> f = add;
```

### 2. lambda

```cpp
std::function<int(int, int)> f = [](int a, int b) {
    return a + b;
};
```

### 3. 带捕获的 lambda

```cpp
int base = 10;

std::function<int(int)> f = [base](int x) {
    return x + base;
};
```

### 4. 函数对象

```cpp
struct Add {
    int operator()(int a, int b) const {
        return a + b;
    }
};

std::function<int(int, int)> f = Add{};
```

这几种“长相不同”的东西，
最后都能被统一塞进：

- `std::function<int(int, int)>`

---

## 它真正统一的是什么

不是“统一代码长相”，
而是：

- **统一调用接口**

只要你保证签名一致：

- 参数类型一致
- 返回类型能对上

那外面用的人就不用关心：

- 里面装的是普通函数
- 还是 lambda
- 还是函数对象

这就是它在工程里的意义。

---

## 一个特别典型的场景：回调

```cpp
#include <functional>
#include <iostream>

void run_twice(const std::function<void()>& task) {
    task();
    task();
}

int main() {
    int x = 0;

    run_twice([&x]() {
        x++;
        std::cout << "x = " << x << "\n";
    });
}
```

这里 `run_twice` 根本不在乎你传进来的是谁，
它只在乎：

- 能不能用 `task()` 调起来

这就是 `std::function` 最常见的使用画面：

- **回调接口**

---

## 它和 C 语言回调类型到底差在哪

先看 C 风格回调最经典的样子：

```c
typedef void (*Callback)(int);
```

这本质上是：

- **函数指针**

也就是说，它能指向的通常是：

- 普通函数
- 静态函数

但它自己**不能天然带状态**。

所以 C 里如果你想让回调“顺手带点上下文”，
通常会额外再传一个：

```c
typedef void (*Callback)(int value, void* user_data);
```

也就是经典的：

- 函数指针 + `void* context`

### `std::function` 和它的核心区别

`std::function<void(int)>` 则是：

- 一个统一的可调用对象包装器

它不仅能装普通函数，
还可以装：

- lambda
- 带捕获的 lambda
- 函数对象

所以它能天然做到：

- **“回调逻辑”和“上下文状态”一起打包**

比如：

```cpp
int base = 10;

std::function<void(int)> cb = [base](int x) {
    std::cout << x + base << "\n";
};
```

这里 `base` 就跟着回调一起进去了，
不需要你再手动传一个 `void* user_data`。

### 一句话对比

- C 回调：更像“函数地址 + 额外上下文指针”
- `std::function`：更像“把行为和状态一起装起来的对象”

---

## 代价和适用场景也不一样

### C 风格回调

优点：

- 很轻
- ABI 稳定
- 容易跨语言 / 跨库边界
- 系统库、C 库、内核接口特别常见

缺点：

- 状态要自己手动塞进 `void*`
- 类型信息弱
- 容易把上下文转错

### `std::function`

优点：

- 表达力更强
- 能直接装带状态的 lambda
- 接口更现代，类型更清楚

缺点：

- 有类型擦除和额外开销
- 不能直接当 C ABI 回调用

---

## 一个你必须知道的限制

如果某个 C API 要的是：

```c
void (*cb)(int, void*)
```

那你**不能直接**把一个 `std::function` 传进去。

因为：

- C API 要的是函数指针
- `std::function` 是 C++ 对象，不是裸函数地址

这时常见做法还是：

1. 传一个普通函数 / 静态函数当桥接函数
2. 再通过 `void* user_data` 把上下文传进去

---

## 还有一个细节：不是所有 lambda 都能当 C 回调

### 不带捕获的 lambda

通常可以转成函数指针：

```cpp
auto f = [](int x) {
    std::cout << x << "\n";
};
```

### 带捕获的 lambda

通常不行：

```cpp
int base = 10;
auto f = [base](int x) {
    std::cout << x + base << "\n";
};
```

因为带捕获 lambda 本质上已经是：

- 带状态的对象

而不是单纯的函数地址了。

所以你也能从这个角度理解：

- C 风格回调更接近“裸函数入口”
- `std::function` / 捕获 lambda 更接近“对象化回调”

---

## 和 lambda 到底是什么关系

你可以这样理解：

- lambda 是一种可调用对象
- `std::function` 是装可调用对象的盒子

所以关系不是：

- 它们互相替代

而是：

- lambda 常常被装进 `std::function`

比如：

```cpp
int base = 100;

std::function<int(int)> f = [base](int x) {
    return x + base;
};
```

---

## 一个你必须知道的点：它是“有代价”的

`std::function` 很方便，
但它不是零成本抽象。

它通常意味着：

1. 类型擦除成本
2. 间接调用成本
3. 某些情况下可能发生额外分配

所以结论不是“别用”，
而是：

- **该用在接口抽象、回调存储时再用**
- **不要默认所有地方都包一层 `std::function`**

---

## 那什么时候不用 `std::function`

这是很重要的一层判断。

### 情况 1：你只是临时把 lambda 传给模板算法

比如：

```cpp
std::sort(v.begin(), v.end(), [](int a, int b) {
    return a > b;
});
```

这里就完全不需要 `std::function`。

因为：

- 算法模板会直接吃掉这个 lambda 的真实类型

### 情况 2：你在写泛型函数，想要零额外包装

这时通常更自然的是模板：

```cpp
template <class F>
void run(F f) {
    f();
}
```

而不是：

```cpp
void run(std::function<void()> f) {
    f();
}
```

### 一句话区分

- 只“调用一下”，优先想模板 / `auto`
- 需要“存起来、统一类型、做接口”，再想 `std::function`

---

## 一个最容易懂的对比

### 模板版本

```cpp
template <class F>
void run(F f) {
    f();
}
```

特点：

- 不需要统一类型
- 编译期决定具体 callable 类型
- 更像“把真实类型原样吃进来”

### `std::function` 版本

```cpp
void run(const std::function<void()>& f) {
    f();
}
```

特点：

- 接口更统一
- 调用方更自由
- 更适合“回调字段 / 统一 API”

---

## 空 `std::function` 是什么情况

`std::function` 也可以是空的：

```cpp
std::function<void()> f;

if (f) {
    f();
}
```

所以在工程里常会先判断：

- 这个回调有没有被设置

这和裸函数指针判空的感觉有点像，
但表达更现代。

---

## 这一页真正该记住的 6 句话

1. `std::function` 是统一包装可调用对象的类型
2. 它统一的是“调用签名”，不是代码写法
3. lambda 常常会被装进 `std::function`
4. 它很适合做回调接口、成员回调字段、回调容器
5. 它是有代价的，不该无脑到处包
6. 只临时调用时更常见的是模板；需要统一存储和接口时再用 `std::function`

---

## 面试够用线

至少要能顺着讲清：

1. `std::function` 是可调用对象的通用包装器
2. 它能装普通函数、lambda、带捕获 lambda、函数对象
3. 它最常见的用途是回调接口和统一类型存储
4. 它背后通常有类型擦除和额外开销
5. 模板和 `std::function` 的取舍，本质是“零开销泛型”对“统一接口抽象”

---

## 下一步最自然接什么

学完这页之后，
下一步最自然的就是：

1. `std::thread`
2. 线程同步：`mutex`、`condition_variable`

因为这时候你会很自然看到：

- lambda 作为线程入口
- `std::function` 作为任务回调
- 智能指针管理异步对象生命周期

这几块会开始真正串起来。

## 相关页面

- [[04-C-Cpp/C++ lambda 入门：本质、捕获与算法配合]]
- [[04-C-Cpp/C++ std thread 入门：线程入口、join 与参数传递]]
- [[04-C-Cpp/C++ 智能指针入门：unique_ptr、shared_ptr 与 weak_ptr]]
- [[04-C-Cpp/C++ 容器与对象语义下一步学习路线]]
