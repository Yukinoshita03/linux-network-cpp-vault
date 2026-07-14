# C++ std thread 入门：线程入口、join 与参数传递

## 先给一句话定义

`std::thread` 可以先理解成：

- **在当前进程里启动一条新的执行流**

它最重要的不是 API 名字，
而是这件事：

- 同一个进程里，会有两段代码并发往前跑

所以从这里开始，
你学的就不再只是语法，
而是：

- 执行流
- 生命周期
- 参数怎么进线程
- 线程什么时候结束

---

## 为什么这一步正好接在 `lambda` / `std::function` 后面

因为线程入口本质上就是：

- **给新线程一个可调用对象**

比如：

```cpp
std::thread t([] {
    std::cout << "hello from thread\n";
});
```

这里你传进去的就是一个 lambda。

所以你前面刚学的：

- lambda
- 捕获
- `std::function`
- 可调用对象带状态

从这里开始都会落地。

---

## 最小例子：启动一个线程

```cpp
#include <iostream>
#include <thread>

int main() {
    std::thread t([] {
        std::cout << "worker thread\n";
    });

    t.join();
}
```

这段代码做了两件事：

1. 创建线程对象 `t`
2. 启动一条新线程去执行 lambda

`join()` 的意思是：

- 主线程在这里等这个子线程跑完

---

## `join` 为什么重要

这是 `std::thread` 最先必须记住的点。

如果一个 `std::thread` 对象还“管着一条活线程”，
你就不能让它直接析构不管。

所以通常你必须二选一：

1. `join()`
2. `detach()`

### `join()`

- 当前线程等待目标线程结束
- 最常见、最稳妥

### `detach()`

- 把线程放飞
- 线程自己在后台跑
- 原来的 `std::thread` 对象不再管理它

对初学阶段先记一句：

- **默认先用 `join`，不要乱 `detach`**

---

## 一个很关键的机制：线程入口拿到的是“可调用对象 + 参数”的副本/转移结果

看这个：

```cpp
int x = 10;

std::thread t([x] {
    std::cout << x << "\n";
});
```

这里不是说线程“回头找主线程里的 `x`”，
而是：

- lambda 自己先捕获了一份 `x`
- `std::thread` 再把这个 lambda 对象拿去当线程入口

所以你前面那句理解可以继续升级成：

- `function` / lambda 的状态，是跟着 callable 对象本身走的
- 启线程时，线程拿到的是这个 callable 对象

---

## 线程参数默认怎么传

看这个：

```cpp
void work(int x) {
    std::cout << x << "\n";
}

std::thread t(work, 42);
```

这可以理解成：

- 新线程会调用 `work(42)`

关键点是：

- `std::thread` 默认会把这些参数按值保存一份

所以你要先记住：

- **默认不是引用传递**

### 更准确一点：不是“默认存右值”

这里特别容易说歪。

`std::thread t(add_one, x);` 里更准确的说法是：

- `std::thread` 默认会把参数**按值保存**
- 对 lvalue 参数，通常是拷一份
- 对你显式 `std::move(...)` 进去的东西，通常是移进去

所以它不是：

- “默认把东西存成右值”

因为：

- **右值 / 左值是表达式类别**
- **拷贝/移动保存是对象构造方式**

这两个不是一回事。

你可以直接把这几种情况记成：

- `x`：按值拷进去
- `std::move(x)`：按值移进去
- `std::ref(x)`：按引用语义包装进去
- `&x`：按指针值传进去

---

## 如果你想传引用，要用 `std::ref`

这是面试和实战都很常见的点。

先纠正一个很容易混的地方：

- **不是“函数参数是 `T&`，就必须写 `std::ref`”**
- `std::ref` 只是在 `std::thread` 这种包装层里，用来保住引用语义

如果你是普通直接调用函数，
该怎么传还是怎么传：

```cpp
void add_one(int& x) {
    x++;
}

int x = 10;
add_one(x);      // 直接正常传 lvalue
```

这里完全不需要 `std::ref`。

真正需要 `std::ref` 的场景是：

- 外面这层 API 不会立刻直接调函数
- 而是先把“函数 + 参数”存起来，再稍后调用

比如：

- `std::thread`
- `std::bind`
- 某些任务队列 / 回调封装

这时它们默认更偏向“按值保存参数”，
所以你才需要显式说：

- 这个别存副本，给我按引用语义保存

```cpp
#include <functional>
#include <iostream>
#include <thread>

void add_one(int& x) {
    x++;
}

int main() {
    int x = 10;
    std::thread t(add_one, std::ref(x));
    t.join();

    std::cout << x << "\n"; // 11
}
```

如果你不写 `std::ref(x)`，
那线程通常拿到的是：

- `x` 的一个副本

而不是外面那个原变量。

所以这件事和 lambda 捕获很像：

- 默认偏向副本
- 真想共享原对象，要显式说

### 那如果直接写裸 `x` 会怎样

比如你这样写：

```cpp
void add_one(int& x) {
    x++;
}

int x = 10;
std::thread t(add_one, x);
```

这里通常不是“线程里改的是副本但还能跑”，
而是：

- **通常直接编译不过**

因为：

1. `std::thread` 会先把 `x` 按值存起来
2. 这样引用属性就丢了，内部更像存了一个 `int`
3. 但 `add_one` 要求的是 `int&`
4. 普通的按值保存结果，不能满足这个“非常量左值引用”参数

所以结论要分两种：

- 函数参数如果是按值 `int`：裸传 `x` 可以，线程拿到副本
- 函数参数如果是 `int&`：裸传 `x` 通常不行，要用 `std::ref(x)`

这也是为什么我前面一直强调：

- `std::ref` 不是“优化写法”
- 而是“显式告诉线程：这里我要按引用传”

### 那如果写 `&x` 呢

比如：

```cpp
int x = 10;
std::thread t(add_one, &x);
```

这要看 `add_one` 的参数类型。

#### 情况 1：`add_one(int& x)`

那通常还是不行。

因为：

- `&x` 的类型是 `int*`
- 但函数要的是 `int&`

也就是说这已经不是“引用语义会不会丢”的问题了，
而是：

- **类型根本对不上**

所以通常直接编译失败。

#### 情况 2：`add_one(int* x)`

那就可以：

```cpp
void add_one(int* x) {
    (*x)++;
}

int x = 10;
std::thread t(add_one, &x);
t.join();
```

这时线程里拿到的是：

- 一个指向 `x` 的指针

然后你在函数体里手动解引用去改原值。

### 所以 `std::ref(x)` 和 `&x` 不是一回事

- `std::ref(x)`：还是按“引用语义”把 `x` 传给形参里的 `T&`
- `&x`：传的是一个真指针，形参也得写成 `T*`

你可以直接把这 3 种写法记成：

- `x`：默认按值
- `std::ref(x)`：按引用语义
- `&x`：按指针语义

---

## `lambda` 线程入口最常见的两种写法

### 1. 值捕获

```cpp
int x = 10;

std::thread t([x] {
    std::cout << x << "\n";
});
```

线程看到的是：

- `x` 的副本

### 2. 引用捕获

```cpp
int x = 10;

std::thread t([&x] {
    x++;
});
t.join();
```

线程改的是：

- 外面那个真 `x`

但这里也开始埋下并发风险：

- 多线程同时碰同一个变量，就可能有数据竞争

这就是下一步为什么要学 `mutex`。

---

## `std::thread` 和 `std::function` 是什么关系

`std::thread` 并不要求你一定传 `std::function`，
它更宽：

- 只要是可调用对象就行

比如直接传 lambda：

```cpp
std::thread t([] {
    std::cout << "run\n";
});
```

也可以先用 `std::function` 装起来再传：

```cpp
std::function<void()> task = [] {
    std::cout << "run\n";
};

std::thread t(task);
```

所以关系是：

- `std::function` 负责统一存 callable
- `std::thread` 负责拿 callable 去开线程

---

## 一个特别好的例子：移动 `unique_ptr` 进线程

```cpp
#include <iostream>
#include <memory>
#include <thread>

int main() {
    auto p = std::make_unique<int>(42);

    std::thread t([ptr = std::move(p)]() mutable {
        std::cout << *ptr << "\n";
    });

    t.join();
}
```

这段代码特别适合你把前面知识串起来：

1. `unique_ptr` 不能拷贝，只能移动
2. lambda 可以捕获状态
3. 这里是把所有权直接移进线程入口对象里
4. 线程结束后，那个捕获进去的 `unique_ptr` 也会正常析构

这就是现代 C++ 很典型的味道。

---

## 一个高频报错：lambda 形参和线程实参没对上

比如这段：

```cpp
std::thread([](int a) {
    std::cout << "Hello from thread!\n";
}).detach();
```

这里错不在：

- 临时 `std::thread` 对象后面接 `.detach()`

这件事本身是可以的。

真正错在：

- 这个 lambda 要求一个 `int` 参数
- 但你启动线程时没有传任何参数

也就是说，线程内部最终想做的是：

```cpp
lambda();
```

但这个 lambda 的真实签名更像：

```cpp
lambda(int)
```

所以根本调不起来。

你可以改成两种合法写法。

### 写法 1：lambda 不要参数

```cpp
std::thread([] {
    std::cout << "Hello from thread!\n";
}).detach();
```

### 写法 2：保留参数，并把实参传进去

```cpp
std::thread([](int a) {
    std::cout << a << "\n";
}, 42).detach();
```

这里的本质规则就是：

- **`std::thread` 里的 callable 和后面跟着的参数，最后必须能组成一次合法函数调用**

也就是它内部得真的能拼成：

```cpp
f(args...)
```

如果形参数量或类型对不上，就会编译失败。

### 还有一个容易顺手踩的点

虽然 `.detach()` 这里语法没错，但演示代码通常更推荐先用 `join()`。

因为 `detach()` 之后：

- 主线程不再等它
- `main` 可能很快结束
- 子线程甚至可能还没来得及输出

所以教学和调试阶段，通常先写：

```cpp
std::thread t([] {
    std::cout << "Hello from thread!\n";
});

t.join();
```

---

## 初学阶段最容易踩的 4 个坑

### 1. 忘记 `join` / `detach`

这是最经典的问题。

对初学阶段，你先记死：

- 线程开了，默认就 `join`

### 2. 以为参数默认就是引用传递

不是。

默认通常是值保存。
想传引用就写：

- `std::ref(x)`

### 3. 在线程里引用捕获局部变量，但变量先死了

这和你前面学的悬空引用是同一个坑，
只是在线程里更危险。

### 4. 多线程同时改同一个变量，却没同步

这会导致数据竞争。

这个坑不是 `thread` 自己解决的，
而是下一步：

- `mutex`

---

## 这一页真正该记住的 6 句话

1. `std::thread` 是在同一进程里启动新的执行流
2. 线程入口本质上是一个可调用对象
3. 默认先用 `join`，不要乱 `detach`
4. `std::thread` 默认更像按值保存参数
5. 真想按引用传参数，通常要用 `std::ref`
6. 一旦多个线程碰同一份数据，就会立刻进入同步问题

---

## 面试够用线

至少要能顺着讲清：

1. `std::thread` 怎么启动线程
2. `join` 和 `detach` 的区别
3. 线程入口可以是函数、lambda、函数对象
4. 参数默认不是裸引用传递，引用常要借助 `std::ref`
5. lambda 捕获状态可以跟着线程入口对象一起进线程
6. 多线程共享数据会引出竞态条件和互斥同步

---

## 下一步最自然接什么

学完 `std::thread` 之后，
下一步最自然就是：

1. `mutex`
2. `lock_guard`
3. `condition_variable`

因为从这里开始，
“线程怎么开”已经不是核心问题了，
“线程怎么安全地一起碰数据”才是核心问题。

## 相关页面

- [[02-Linux/POSIX pthread 入门：pthread_create、属性与参数传递]]
- [[04-C-Cpp/C++ 线程深入理解：执行流、共享内存与生命周期]]
- [[04-C-Cpp/C++ 线程入门总览：执行流、同步与学习顺序]]
- [[04-C-Cpp/C++ lambda 入门：本质、捕获与算法配合]]
- [[04-C-Cpp/C++ std function 入门：lambda、回调与统一可调用类型]]
- [[04-C-Cpp/C++ 智能指针入门：unique_ptr、shared_ptr 与 weak_ptr]]
- [[04-C-Cpp/C++ 容器与对象语义下一步学习路线]]
- [[04-C-Cpp/C++ 线程同步入门：竞态条件、mutex 与 lock_guard]]
