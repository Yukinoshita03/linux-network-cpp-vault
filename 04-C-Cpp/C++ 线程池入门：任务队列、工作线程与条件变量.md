# `C++ 线程池` 入门：任务队列、工作线程与条件变量

## 为什么要有线程池

如果你每来一个任务就新建一个线程，
通常会有几个问题：

1. 线程创建和销毁有成本
2. 线程太多会让调度压力变大
3. 并发数失控时，程序更容易抖
4. 任务来了很多时，没法平稳排队

线程池就是为了解决这些问题。

它的核心思路是：

- **提前准备好一组工作线程**
- **把任务放进队列**
- **让工作线程循环取任务执行**

所以线程池不是“再发明一种线程”。
它是一个**任务调度模型**。

### 更本质一点

线程池的目的通常不是：

- 让单个任务更快

而是：

- **把并发数量控制住**
- **把线程创建/销毁成本摊平**
- **把突发任务放进队列里平稳处理**
- **让系统的资源使用更可预测**

所以它更像一个：

- **稳定器**

而不是单纯的“加速器”。

---

## 一句话定义

线程池可以先理解成：

- **固定数量的工作线程 + 共享任务队列 + 条件变量唤醒**

它把“线程”和“任务”解耦了：

- 提交任务的人，不直接管线程怎么跑
- 工作线程，不直接管任务是谁提交的

---

## 这个模型到底在干什么

你可以先把它想成这条流水线：

1. 外部不断 `submit` 任务
2. 任务进入共享队列
3. 队列为空时，工作线程睡眠
4. 有新任务时，`condition_variable` 唤醒一个工作线程
5. 工作线程取出任务并执行
6. 任务执行完，继续回到队列前面等待下一项

这就把：

- `thread`
- `mutex`
- `condition_variable`
- `queue`

串成了一个完整工程模型。

---

## 线程池里每个组件负责什么

### 1. 工作线程

它们是真正执行任务的人。

通常是固定数量，比如：

- 4 个
- 8 个
- 跟 CPU 核数差不多

### 2. 任务队列

它保存待执行任务。

最常见的数据结构是：

```cpp
std::queue<std::function<void()>> tasks;
```

因为线程池里的任务通常只需要：

- “无参数、无返回值地执行一段逻辑”

### 3. `mutex`

它保护任务队列和停止标志。

因为多个线程会同时：

- 往队列里塞任务
- 从队列里取任务
- 修改停止状态

### 4. `condition_variable`

它负责让工作线程在“没任务”时睡下，
有任务时再醒来。

所以它不是保护任务队列本身，
而是协调：

- **队列空时睡**
- **队列非空时醒**

### 5. 停止标志

线程池需要知道什么时候该收工。

通常会有一个：

```cpp
bool stop = false;
```

用来表示：

- 还能不能继续接收/执行任务

---

## 最小可理解代码骨架

```cpp
#include <condition_variable>
#include <functional>
#include <mutex>
#include <queue>
#include <stdexcept>
#include <thread>
#include <vector>

class ThreadPool {
public:
    explicit ThreadPool(size_t n) {
        for (size_t i = 0; i < n; ++i) {
            workers.emplace_back([this] {
                worker_loop();
            });
        }
    }

    ~ThreadPool() {
        {
            std::lock_guard<std::mutex> lock(m);
            stop = true;
        }
        cv.notify_all();

        for (auto& t : workers) {
            if (t.joinable()) {
                t.join();
            }
        }
    }

    void submit(std::function<void()> task) {
        {
            std::lock_guard<std::mutex> lock(m);
            if (stop) {
                throw std::runtime_error("thread pool stopped");
            }
            tasks.push(std::move(task));
        }
        cv.notify_one();
    }

private:
    void worker_loop() {
        while (true) {
            std::function<void()> task;

            {
                std::unique_lock<std::mutex> lock(m);
                cv.wait(lock, [this] {
                    return stop || !tasks.empty();
                });

                if (stop && tasks.empty()) {
                    return;
                }

                task = std::move(tasks.front());
                tasks.pop();
            }

            try {
                task();
            } catch (...) {
                // 生产代码里这里通常会记录日志
            }
        }
    }

private:
    std::mutex m;
    std::condition_variable cv;
    std::queue<std::function<void()>> tasks;
    std::vector<std::thread> workers;
    bool stop = false;
};
```

这段代码先不要背，
先看它的控制流。

---

## 工作线程是怎么工作的

工作线程逻辑本质上就是一个循环：

1. 先拿锁
2. 看队列里有没有任务
3. 没任务就睡
4. 有任务就取一个出来
5. 释放锁
6. 执行任务
7. 回到第一步

最关键的一点是：

- **任务执行时不持有队列锁**

否则：

- 线程执行任务的时候，别人就没法继续提交任务了
- 并发度会很差

所以线程池的套路非常典型：

- **锁只保护“取任务 / 放任务 / 改状态”这小段**
- **真正干活的时候别拿着锁**

---

## `condition_variable` 在这里具体干嘛

工作线程在这句上睡：

```cpp
cv.wait(lock, [this] {
    return stop || !tasks.empty();
});
```

它的含义是：

- 只要还没停，并且队列还是空的，就继续睡
- 一旦有任务进来了，或者线程池要停止了，就醒来

所以线程池里的条件变量不是“可有可无的辅助”。
它是这个模型的核心之一。

没有它的话，你只能：

- 忙等
- 白烧 CPU

这就失去线程池的意义了。

---

## 为什么 `stop || !tasks.empty()` 很重要

这个判断不是装饰。
它是线程池能平稳退出的关键。

### `!tasks.empty()`

表示：

- 还有活要干

### `stop`

表示：

- 线程池准备收工了

这两个条件要一起看，
因为线程池关闭时通常有两种策略：

1. 立刻停止，不再接收新任务
2. 先把队列里已有任务做完，再退出

上面这份代码走的是更温和的方式：

- **先停止接收新任务**
- **把已有任务做完**
- **再退出工作线程**

---

## 为什么任务要用 `std::function<void()>`

因为线程池本质上要接收“任何可调用对象”。

比如：

- 普通函数
- lambda
- 函数对象

这些都能被统一包装成：

```cpp
std::function<void()>
```

这样线程池的接口就很干净：

- 你提交一个“任务”
- 线程池把它放进队列
- 工作线程拿出来执行

这也是为什么你前面先学 `std::function` 和 `lambda` 很有用。

---

## 线程池和生产者消费者是什么关系

线程池本质上就是一个更工程化的生产者消费者模型。

### 生产者

外部提交任务的人就是生产者。

### 消费者

工作线程就是消费者。

### 队列

任务队列就是共享缓冲区。

### 条件变量

队列空的时候让消费者睡，队列有任务时叫醒它。

所以线程池可以直接看成：

- **生产者消费者模型 + 固定工作线程**

---

## 为什么任务执行不能包在锁里

这是线程池里最重要的工程习惯之一。

如果你这样写：

```cpp
std::lock_guard<std::mutex> lock(m);
task();
```

那就意味着：

- 任务执行期间，队列锁一直被占着

问题会很多：

1. 其他线程提交任务会被堵住
2. 工作线程之间会互相抢不到活
3. 锁竞争会变严重

所以正确做法是：

1. 先在锁内把任务从队列里取出来
2. 立刻放锁
3. 再执行任务

这就是“锁只保护共享状态，不保护业务执行”的典型例子。

---

## 最容易踩的几个坑

### 1. 忘记停止标志

如果线程池退出时没有 `stop`，
工作线程就不知道什么时候该停。

### 2. 在持锁时执行任务

这会把并发度拉垮，
还可能把别的线程堵死。

### 3. 只 `notify_one` 不处理退出

关闭线程池时通常要 `notify_all()`，
让所有工作线程都醒过来检查 `stop`。

### 4. 任务抛异常没处理

如果线程里的异常没抓住，
可能直接导致 `std::terminate`。

### 5. 任务里等任务

如果一个任务又去等同一个线程池里的别的任务，
而池子里的工作线程又不够，
就可能形成饥饿甚至死锁。

这个坑很常见。

---

## 这类模型的工程价值

线程池的核心价值不是“我也会开线程”。

它的价值是：

- **把任务提交和线程执行解耦**
- **把并发数量控制在一个可预期范围**
- **把线程创建/销毁成本摊平**
- **让系统在任务突发时不至于乱掉**

所以它在服务端、后台任务、日志写入、异步工作队列里都很常见。

---

## 面试够用回答

如果别人问：

“线程池是怎么工作的？”

你可以这样答：

线程池本质上是一个固定数量工作线程的任务调度模型。外部线程把任务提交到共享队列里，队列空时工作线程通过 `condition_variable` 睡眠，有任务时被唤醒后从队列里取出任务执行。队列和停止状态通常由 `mutex` 保护，线程池关闭时会设置停止标志并 `notify_all` 唤醒所有工作线程再 `join` 回收。

---

## 一句话记忆

线程池就是：

- **固定的工作线程负责消费任务队列，条件变量负责在没任务时睡、有任务时醒**

## 下一步

如果你想继续往下走，下一层最自然的是：

1. `packaged_task` 和 `future`
2. 线程池如何返回结果
3. 更完整的异常传播与取消语义

## 相关页面

- [[04-C-Cpp/C++ 线程入门总览：执行流、同步与学习顺序]]
- [[04-C-Cpp/C++ 线程同步入门：竞态条件、mutex 与 lock_guard]]
- [[04-C-Cpp/C++ condition_variable 入门：等待、唤醒与生产者消费者]]
- [[04-C-Cpp/C++ 生产者消费者模型入门：mutex、condition_variable 与队列协作]]
- [[04-C-Cpp/C++ 死锁入门：锁顺序、活锁与常见并发坑]]
- [[04-C-Cpp/C++ std function 入门：lambda、回调与统一可调用类型]]
- [[04-C-Cpp/C++ lambda 入门：本质、捕获与算法配合]]
## 如果每个任务的参数都不一样怎么办

线程池通常不会让任务队列直接存“不同签名”的函数。更常见的做法是：
先把参数绑定进一个可调用对象，再把它统一收敛成 `void()`。

最常见的是 lambda 捕获：

```cpp
pool.submit([x, y] {
    do_work(x, y);
});
```

这里 `x`、`y` 已经跟这一次任务绑定好了，
所以队列里看到的仍然是一个无参数任务。

如果你想改外部状态，可以用引用捕获或 `std::ref`，
但要保证被引用对象的生命周期比任务长：

```cpp
pool.submit([&counter] {
    ++counter;
});
```

更工程化的线程池接口一般会写成模板：

```cpp
template<class F, class... Args>
auto submit(F&& f, Args&&... args)
    -> std::future<std::invoke_result_t<F, Args...>>;
```

接口内部再把 `f` 和 `args...` 绑定成一个任务。
如果还要返回值，通常就接 `packaged_task + future`。
相比 `std::bind`，lambda 捕获通常更直观。

## 为什么这里要保留右值

这里的 `Args&&...` 不是在“强行用右值”，而是在**保留参数本来的值类别**。

- 传进来的是临时对象，就把它当右值处理，优先移动进任务里
- 传进来的是命名变量，就把它当左值处理，按原语义使用

这样做的好处是：

1. 少拷贝，尤其是 `string`、`vector`、`unique_ptr` 这种对象
2. 支持只能移动、不能拷贝的类型
3. 任务一旦入队，通常就是“一次性消费”，把资源搬进任务里最自然

如果你不用 `std::forward`，而是直接写成普通参数传递，
很多本来可以移动的对象就会退化成拷贝。

一个直观例子：

```cpp
std::unique_ptr<int> p = std::make_unique<int>(10);
pool.submit(use_ptr, std::move(p));
```

这里 `std::move(p)` 的意思不是“现在就移动”，而是告诉模板：
“这个对象可以被当成将死资源处理，尽量把所有权移进任务里”。

所以这套写法的核心不是“右值更快”这么粗，
而是：**线程池要把一次性的输入，安全地转成一次性的任务**。

一句话再压缩一下：

- 右值不是目的
- “把参数一次性搬进任务对象”才是目的
- `std::forward` 只是把这个意图准确传给库和编译器

## 这段 `submit` 代码在干嘛

```cpp
template <class F, class... Args>
auto submit(F&& f, Args&&... args) {
    std::lock_guard<std::mutex> lock(g_mutex);
    q.push(std::bind(std::forward<F>(f), std::forward<Args>(args)...));
}
```

这段代码做的是：

1. `submit` 接收一个可调用对象 `f`，再接收任意个参数 `args...`
2. `std::forward` 保留参数是左值还是右值
3. `std::bind(...)` 把 `f` 和这些参数提前绑成一个新任务
4. `q.push(...)` 把这个“绑好的任务”塞进队列
5. `lock_guard` 只负责保护这次入队，防止多个线程同时改队列

它的本质就是把：

```cpp
submit(print_sum, 1, 2);
```

变成一个以后可以直接执行的任务对象，等工作线程取出来时只需要调用：

```cpp
task();
```

要注意两点：

- `std::bind` 默认会把参数按值存起来，所以如果你要真引用外部变量，通常要传 `std::ref(x)`
- 这段代码只是在“收任务”，如果要做完整线程池，通常还要在入队后 `notify_one()`，而且如果任务有返回值，往往还要配 `future`

### 为什么绑定完以后队列里只需要 `void()`

`std::bind(...)` 的结果，不再是“原始函数 + 原始参数列表”，
而是一个**已经把参数塞进去的可调用对象**。

比如：

```cpp
auto task = std::bind(print_sum, 1, 2);
task();
```

这里 `task()` 之所以只要空参数，是因为 `1` 和 `2` 已经在 `bind` 时被记住了。
后面执行时，真正调用的是：

```cpp
print_sum(1, 2);
```

所以线程池的任务队列通常只存统一签名，比如：

```cpp
std::queue<std::function<void()>> q;
```

原因很简单：

- 队列里要装很多不同任务
- 每个任务原始函数签名都可能不同
- 但容器里每一项必须是同一种类型

`std::function<void()>` 就是这个“统一类型”。
工作线程从队列里拿出来以后，不关心原来是什么函数，只管调用 `task()`。

如果你写成 `std::function<void(F&&, Args&&...)>`，那每个任务的 `F/Args...` 都可能不同，
队列就没法变成一个真正统一的容器。

### 最短理解

- `bind` 先把参数塞进任务里
- 任务一旦绑定完，就不再需要原始参数
- 队列只需要存“以后直接能 `task()` 的东西”

## 按这种风格写一个能跑的最小线程池

```cpp
#include <condition_variable>
#include <functional>
#include <mutex>
#include <queue>
#include <thread>
#include <utility>
#include <vector>

class ThreadPool {
public:
    explicit ThreadPool(size_t n) : stop_(false) {
        for (size_t i = 0; i < n; ++i) {
            workers_.emplace_back([this] {
                while (true) {
                    std::function<void()> task;

                    {
                        std::unique_lock<std::mutex> lock(mutex_);
                        cv_.wait(lock, [this] {
                            return stop_ || !tasks_.empty();
                        });

                        if (stop_ && tasks_.empty()) {
                            return;
                        }

                        task = std::move(tasks_.front());
                        tasks_.pop();
                    }

                    task();
                }
            });
        }
    }

    ~ThreadPool() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            stop_ = true;
        }
        cv_.notify_all();

        for (auto& t : workers_) {
            if (t.joinable()) {
                t.join();
            }
        }
    }

    template <class F, class... Args>
    void submit(F&& f, Args&&... args) {
        auto task = std::bind(
            std::forward<F>(f),
            std::forward<Args>(args)...);

        {
            std::lock_guard<std::mutex> lock(mutex_);
            tasks_.push(std::move(task));
        }

        cv_.notify_one();
    }

private:
    std::mutex mutex_;
    std::condition_variable cv_;
    std::queue<std::function<void()>> tasks_;
    std::vector<std::thread> workers_;
    bool stop_;
};
```

这版保留了你原来想要的核心结构：

- 一个任务队列
- 一把互斥锁
- 一个条件变量
- 多个工作线程循环取任务

但修掉了这几个关键问题：

- 队列必须存 `std::function<void()>`，不能存带 `F/Args...` 的签名
- `submit` 里入队后必须 `notify_one()`
- worker 取出任务后必须真的执行 `task()`
- `wait` 里的 lambda 要捕获 `this`，否则访问不到成员
- 线程池析构时要有 `stop_ + notify_all + join`，不能直接全 `detach`

### `cv.wait(lock, [this] { return stop_ || !tasks_.empty(); })` 是什么意思

这句要拆开看：

- `[this]`：把当前对象指针捕获进 lambda
- `stop_` 和 `tasks_` 都是成员变量，所以 lambda 要靠 `this` 才能访问它们
- `return stop_ || !tasks_.empty();`：这是“什么时候可以不睡了”的条件

也就是：

- 如果线程池要关闭了，醒来
- 如果任务队列里有任务了，醒来
- 否则继续睡

所以这不是“执行某个回调”，而是 `condition_variable` 在每次醒来后都要重新检查的谓词。

## 走到这里，下一步最合适学什么

如果你已经把：

- `mutex`
- `condition_variable`
- 任务队列
- worker loop
- `submit`

这条线看明白了，
那下一步最自然的不是再开新并发概念，
而是把线程池补完整：

1. `packaged_task`
2. `future`
3. 线程池如何返回结果

因为你现在这版线程池只能“把任务跑掉”，
但工程里更常见的问题是：

- 我把任务扔进线程池以后，结果怎么拿回来？
- 任务里面抛异常了，调用方怎么感知？

这正好会把你前面学过的：

- 可调用对象
- 移动语义
- 完美转发

和并发模型真正串起来。

## `future` 和 `packaged_task` 到底是干嘛的

如果前面的线程池只是这样：

```cpp
std::queue<std::function<void()>> tasks;
```

那它只能做到一件事：

- 把任务跑掉

但它还做不到：

- 任务算出来的结果怎么回到提交者手里
- 任务里面抛异常了，提交者怎么知道

这就是 `future` 和 `packaged_task` 出场的原因。

### 1. `future` 是“以后再拿结果的票据”

你可以先把 `future<T>` 理解成：

- 现在结果还没好
- 但我手里先拿到一个“将来可以取结果”的对象

最常见的用法是：

```cpp
std::future<int> f = ...;
int x = f.get();
```

`get()` 的意思是：

- 如果结果已经准备好了，就直接拿出来
- 如果结果还没准备好，就在这里等

所以 `future` 的本质是：

- **结果通道**

它让“提交任务的人”和“真正执行任务的线程”之间，
多了一条正规的结果传递路径。

### 2. `packaged_task` 是“把函数和结果通道绑在一起”

`packaged_task<R()>` 可以先理解成：

- 一个可调用任务
- 但它内部顺手带着一条 future 对应的结果槽

比如：

```cpp
std::packaged_task<int()> task([] {
    return 42;
});

std::future<int> f = task.get_future();
task();
int x = f.get();
```

这段流程是：

1. 先把一个函数包装成 `packaged_task`
2. 从它身上拿到 `future`
3. 真正执行 `task()`
4. 返回值会自动写进 `future` 对应的共享状态
5. 外面再用 `f.get()` 拿结果

所以 `packaged_task` 干的事其实是：

- **把“可执行逻辑” 和 “结果存放位置”绑在一起**

### `get_future()` 是拿来干嘛的

`get_future()` 做的不是“执行任务”，
而是：

- 从这个 `packaged_task` 身上把结果通道拿出来

也就是说：

- `task()` 负责将来去算
- `task.get_future()` 负责先把“将来去哪拿结果”这件事交给你

比如：

```cpp
std::packaged_task<int()> task([] {
    return 42;
});

auto f = task.get_future();
```

这时：

- `task` 还没执行
- 但 `f` 已经提前拿到了

后面只要有人执行：

```cpp
task();
```

那这个 `task` 的返回值就会被放进 `f` 对应的共享状态里，
外面就可以再：

```cpp
int x = f.get();
```

所以 `get_future()` 的本质就是：

- **先把取结果的句柄领走**

### `packaged_task` 自己的作用到底是什么

如果只看一句话：

- `packaged_task` 就是**带返回值出口的任务包装器**

它做的事不是“保存结果给你看”，
也不是“负责等待结果”，
而是：

1. 包住一段可调用逻辑
2. 在这段逻辑执行完时，把返回值或异常自动写进共享状态

然后：

- `future` 去连这个共享状态
- 调用方以后再通过 `get()` 取结果

所以三者分工其实很清楚：

- `packaged_task`：负责执行时写结果
- `get_future()`：负责把结果句柄交出去
- `future`：负责以后等待并取回结果

如果没有 `packaged_task`，
线程池就得自己手写一套：

- 结果存哪
- 异常怎么存
- 调用方怎么拿

而 `packaged_task` 已经把这套机制标准化了。

### 3. 它们在线程池里怎么配合

在线程池场景里，通常不是直接把原始函数丢进队列，
而是：

1. 先把函数和参数包装成一个 `packaged_task`
2. 再拿到对应的 `future`
3. 把这个任务塞进线程池队列
4. worker 线程执行任务
5. 调用方以后通过 `future.get()` 拿结果

你可以把它理解成：

- `packaged_task` 负责“把任务和结果出口焊死”
- `future` 负责“在外面等结果”

### 4. 为什么这两个东西很适合线程池

因为线程池天然有一个问题：

- 提交任务的人和执行任务的人不是同一个线程

那就必须补一条机制，解决：

- 结果往哪放
- 异常怎么传回来
- 外面什么时候来取

`packaged_task + future` 就是在标准库里专门解决这个问题的。

### 5. 一句话收住

- `future`：以后取结果的句柄
- `packaged_task`：带结果通道的任务包装器

### 6. 面试时怎么讲这两个接口

你可以直接按这个顺序说：

1. `future` 是结果句柄，不负责计算，只负责等结果和取结果。
2. `packaged_task` 是任务包装器，把一个可调用对象和“结果共享状态”绑在一起。
3. `get_future()` 是先把这个结果句柄拿出来，交给提交任务的人。
4. worker 真正执行 `task()` 时，返回值或异常会被写入共享状态。
5. 外面最后用 `future.get()` 取结果；如果任务里抛了异常，这里会重新抛出来。

所以它们的关系不是并列，而是分工：

- `packaged_task` 负责“生产结果”
- `future` 负责“消费结果”
- `get_future()` 负责“提前把结果入口发出去”

### 7. 一个最短的线程池提交模型

```cpp
template <class F, class... Args>
auto submit(F&& f, Args&&... args) {
    using R = std::invoke_result_t<F, Args...>;
    auto task = std::make_shared<std::packaged_task<R()>>(
        std::bind(std::forward<F>(f), std::forward<Args>(args)...)
    );

    std::future<R> ret = task->get_future();
    {
        std::lock_guard<std::mutex> lock(mutex_);
        tasks_.emplace([task] {
            (*task)();
        });
    }
    cv_.notify_one();
    return ret;
}
```

这里的关键点是：

- 队列里存的是 `void()` 任务
- 真正的返回值走 `future`
- `packaged_task` 只负责把“执行”和“结果出口”连起来

### 8. `task()` 会不会开线程

不会。`std::packaged_task` 只是一个可调用对象包装器，不创建线程，也不负责调度。

```cpp
std::packaged_task<int()> task([] {
    return 42;
});

auto result = task.get_future();
task();
```

最后一行的 `task()` 和普通函数调用一样：它在**当前线程**执行 lambda，算出 `42` 后把结果写进共享状态。由于结果已经就绪，后续 `result.get()` 不会阻塞。

要异步执行，必须由 `std::thread` 或线程池来调用它：

```cpp
std::thread worker(std::move(task));
int value = result.get();
worker.join();
```

这里是 `std::thread` 创建了新线程；它接管并在新线程里调用 `task()`。`std::move(task)` 是必须的，因为 `packaged_task` 不可复制，只能转移给线程或任务队列。

可以用下面这句话区分职责：

- `packaged_task`：任务怎么执行完并写回结果
- `std::thread` / 线程池：任务在哪个线程、什么时候执行
- `future.get()`：结果未就绪时等待，结果就绪后取回

### 9. 一个可运行的最小线程池（C++17）

```cpp
#include <condition_variable>
#include <functional>
#include <future>
#include <iostream>
#include <memory>
#include <mutex>
#include <queue>
#include <stdexcept>
#include <thread>
#include <type_traits>
#include <utility>
#include <vector>

class ThreadPool {
public:
    explicit ThreadPool(std::size_t thread_count) {
        if (thread_count == 0) {
            throw std::invalid_argument("thread_count must be positive");
        }

        workers_.reserve(thread_count);
        for (std::size_t index = 0; index < thread_count; ++index) {
            workers_.emplace_back([this] { worker_loop(); });
        }
    }

    ~ThreadPool() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            stop_ = true;
        }
        cv_.notify_all();

        for (std::thread& worker : workers_) {
            worker.join();
        }
    }

    ThreadPool(const ThreadPool&) = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;

    template <class F, class... Args>
    auto submit(F&& function, Args&&... args)
        -> std::future<std::invoke_result_t<F, Args...>> {
        using Result = std::invoke_result_t<F, Args...>;

        auto task = std::make_shared<std::packaged_task<Result()>>(
            std::bind(std::forward<F>(function), std::forward<Args>(args)...));

        std::future<Result> result = task->get_future();

        {
            std::lock_guard<std::mutex> lock(mutex_);
            if (stop_) {
                throw std::runtime_error("submit on stopped ThreadPool");
            }

            tasks_.emplace([task] {
                (*task)();
            });
        }
        cv_.notify_one();
        return result;
    }

private:
    void worker_loop() {
        while (true) {
            std::function<void()> task;

            {
                std::unique_lock<std::mutex> lock(mutex_);
                cv_.wait(lock, [this] {
                    return stop_ || !tasks_.empty();
                });

                if (stop_ && tasks_.empty()) {
                    return;
                }

                task = std::move(tasks_.front());
                tasks_.pop();
            }

            task();
        }
    }

    std::mutex mutex_;
    std::condition_variable cv_;
    std::queue<std::function<void()>> tasks_;
    std::vector<std::thread> workers_;
    bool stop_ = false;
};

int main() {
    ThreadPool pool(2);

    std::future<int> answer = pool.submit([](int left, int right) {
        return left + right;
    }, 20, 22);

    std::cout << answer.get() << '\n';
}
```

执行轨迹：

1. `submit` 把函数和参数绑成一个 `packaged_task<Result()>`。
2. `get_future()` 把结果句柄返回给调用方。
3. 队列只保存无参数的 `void()` 任务；worker 取出后调用 `task()`。
4. `packaged_task` 把返回值或异常写进共享状态，调用方的 `future.get()` 再取出它。
5. 析构时设置 `stop_`，唤醒所有 worker；worker 清空已入队任务后退出，最后由 `join()` 回收线程。

为什么要用 `shared_ptr<packaged_task<...>>`：`packaged_task` 是只可移动对象，但 C++17 的 `std::function<void()>` 要求内部目标可复制。lambda 捕获一个可复制的 `shared_ptr`，才能安全塞进任务队列。

### 10. 面试手写简化版

面试时可以把 `submit` 简化成“只收无参数任务”。有参数时让调用方用 lambda 捕获，避免把重点耗在模板参数转发上。

```cpp
#include <condition_variable>
#include <functional>
#include <future>
#include <memory>
#include <mutex>
#include <queue>
#include <thread>
#include <vector>

class ThreadPool {
public:
    explicit ThreadPool(int count) {
        for (int index = 0; index < count; ++index) {
            workers_.emplace_back([this] { worker(); });
        }
    }

    ~ThreadPool() {
        {
            std::lock_guard<std::mutex> lock(mutex_);
            stop_ = true;
        }
        cv_.notify_all();
        for (auto& thread : workers_) {
            thread.join();
        }
    }

    template <class F>
    auto submit(F function) -> std::future<decltype(function())> {
        using Result = decltype(function());
        auto task = std::make_shared<std::packaged_task<Result()>>(
            std::move(function));
        auto result = task->get_future();

        {
            std::lock_guard<std::mutex> lock(mutex_);
            tasks_.push([task] { (*task)(); });
        }
        cv_.notify_one();
        return result;
    }

private:
    void worker() {
        while (true) {
            std::function<void()> task;
            {
                std::unique_lock<std::mutex> lock(mutex_);
                cv_.wait(lock, [this] {
                    return stop_ || !tasks_.empty();
                });
                if (stop_ && tasks_.empty()) {
                    return;
                }
                task = std::move(tasks_.front());
                tasks_.pop();
            }
            task();
        }
    }

    std::mutex mutex_;
    std::condition_variable cv_;
    std::queue<std::function<void()>> tasks_;
    std::vector<std::thread> workers_;
    bool stop_ = false;
};
```

调用时：

```cpp
ThreadPool pool(2);
auto future = pool.submit([left = 20, right = 22] {
    return left + right;
});
int answer = future.get();
```

### 11. `decltype` 在 `submit` 里做什么

`decltype(表达式)` 是让编译器在编译期推导“这个表达式的类型”，不会执行表达式。

```cpp
int value = 10;
decltype(value) other = 20; // decltype(value) 是 int
```

在线程池里：

```cpp
template <class F>
auto submit(F function) -> std::future<decltype(function())>;
```

`function()` 只是被放在 `decltype` 中供编译器观察，不会真的调用。假设传入：

```cpp
[] { return 42; }
```

那么 `decltype(function())` 就是 `int`，整个返回类型就是 `std::future<int>`。同一个类型也通过下面的别名复用：

```cpp
using Result = decltype(function());
```

面试先记住：`auto` 是让编译器根据右侧初始化表达式推导变量类型；`decltype(expr)` 是你主动询问某个表达式的类型。`decltype(function())` 常用于泛型代码中推导调用结果类型。

### 12. 函数声明里的 `->` 是什么

```cpp
auto submit(F function) -> std::future<decltype(function())>;
```

`-> 类型` 叫尾置返回类型，等价于把返回类型写在函数名前面。例如：

```cpp
auto add(int left, int right) -> int;
// 等价于
int add(int left, int right);
```

线程池中要写尾置返回类型，是因为返回类型依赖参数 `function`：

```cpp
decltype(function())
```

只有参数列表写完后，`function` 这个名字才可用。因此不能写成下面这样：

```cpp
// 错误：此处 function 还没有被声明
std::future<decltype(function())> submit(F function);
```

假设 `function()` 的结果是 `int`，那么这份声明在实例化后相当于：

```cpp
std::future<int> submit(...);
```

### 13. `using Result = ...` 为什么这样写

```cpp
using Result = decltype(function());
```

`using` 在这里是类型别名声明：给右侧的类型取一个新名字，不创建对象，也不执行代码。

假设提交的函数返回 `int`，编译器会把它理解为：

```cpp
using Result = int;
```

于是后面的代码就更易读：

```cpp
std::packaged_task<Result()> // 实际是 std::packaged_task<int()>
std::future<Result>          // 实际是 std::future<int>
```

不写别名也能工作，只是会反复出现很长的类型：

```cpp
std::packaged_task<decltype(function())()>
```

它等价于旧写法：

```cpp
typedef decltype(function()) Result;
```

现代 C++ 更常用 `using`，因为语序更自然，而且可以定义别名模板：

```cpp
template <class T>
using IntMap = std::map<int, T>;
```

### 14. 线程池之后的学习顺序

当前最适合接着学的是 `std::promise`。它和 `future` 共享同一个结果通道，但不再由 `packaged_task` 自动写入结果，而是由程序员手动调用 `set_value()` 或 `set_exception()`。

建议顺序：

1. `std::promise`：理解谁是结果生产者，谁是结果消费者。
2. `std::async`：理解标准库如何把“启动任务 + 返回 future”打包起来，以及 `std::launch::async` 与 `std::launch::deferred` 的区别。
3. 回头手写线程池：给当前简化版补“拒绝停止后提交任务”和“任务异常通过 future 传回”的测试。
4. `std::atomic`：学习无锁的基础边界、原子读写和 `memory_order` 的入门模型。
5. 并发常见坑：死锁、数据竞争、虚假唤醒、对象生命周期与 `detach()` 风险。
