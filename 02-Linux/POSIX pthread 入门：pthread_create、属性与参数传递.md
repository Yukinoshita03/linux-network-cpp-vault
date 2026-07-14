# POSIX pthread 入门：pthread_create、属性与参数传递

## 一句话

`pthread_create` 在当前进程中创建一条新的线程执行流，并让新线程从指定的入口函数开始执行。

它和 C++ 的 `std::thread` 思想一致，但接口更接近 Linux/POSIX 的底层模型：

- 用 `pthread_t` 保存线程标识
- 用 `pthread_attr_t` 描述创建属性
- 用函数指针指定线程入口
- 用 `void*` 统一传递参数和结果

---

## `pthread_create` 的函数原型

```cpp
int pthread_create(
    pthread_t* thread,
    const pthread_attr_t* attr,
    void* (*start_routine)(void*),
    void* arg
);
```

创建成功后，可以把它理解成：

```text
主线程继续执行
        └── 新线程执行 start_routine(arg)
```

调用返回 `0` 表示创建成功；返回非零值表示错误码。pthread 接口通常直接返回错误码，不能简单套用“失败返回 `-1`，再看 `errno`”的习惯。

---

## 四个参数分别做什么

### `pthread_t* thread`

用于接收新线程的标识：

```cpp
pthread_t thread;
pthread_create(&thread, nullptr, worker, nullptr);
```

`pthread_t` 更接近线程句柄或标识，不是线程执行流本身。

### `const pthread_attr_t* attr`

指向线程创建属性对象。不需要自定义属性时直接传 `nullptr`，表示使用默认配置：

```cpp
pthread_create(&thread, nullptr, worker, nullptr);
```

需要自定义时，必须先初始化属性对象：

```cpp
pthread_attr_t attr;
pthread_attr_init(&attr);

pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_JOINABLE);
pthread_attr_setstacksize(&attr, 1024 * 1024);

pthread_create(&thread, &attr, worker, nullptr);
pthread_attr_destroy(&attr);
```

`pthread_attr_t` 是 POSIX 定义的**不透明属性类型**。程序不应依赖它内部究竟是结构体、联合体还是其他实现形式，只能通过 `pthread_attr_*` 系列函数操作。

这里要区分两个动作：

```text
pthread_attr_t attr;       声明对象并提供存储空间
pthread_attr_init(&attr);  初始化为合法的默认属性状态
```

常见属性包括：

- joinable 或 detached
- 线程栈大小
- 栈保护区大小
- 调度策略和优先级

`pthread_attr_destroy` 销毁的是属性对象内部资源，不会终止已经创建的线程。

### `void* (*start_routine)(void*)`

这是一个函数指针，指向新线程要执行的入口函数。

从变量名向外读：

```text
start_routine
→ 是一个指针
→ 指向接收 void* 参数的函数
→ 该函数返回 void*
```

可以用类型别名写得更直观：

```cpp
using ThreadFunction = void* (*)(void*);
```

因此入口函数必须符合统一签名：

```cpp
void* worker(void* data) {
    return nullptr;
}
```

不能直接写成 `void worker(int value)`，因为它与 pthread 要求保存和调用的函数指针类型不匹配。

### `void* arg`

这是传给入口函数的通用对象指针。pthread 不知道它实际指向 `int`、结构体还是其他对象，只负责原样调用：

```cpp
start_routine(arg);
```

注意术语：

```text
void   表示没有值
void*  是通用对象指针，可以保存不同对象的地址
```

`void*` 不保留具体类型信息，所以在线程入口中必须转换回真实类型：

```cpp
void* worker(void* data) {
    int* value = static_cast<int*>(data);
    std::printf("%d\n", *value);
    return nullptr;
}
```

需要传多个参数时，用结构体打包：

```cpp
struct Task {
    int id;
    const char* name;
};

void* worker(void* data) {
    Task* task = static_cast<Task*>(data);
    std::printf("%d: %s\n", task->id, task->name);
    return nullptr;
}
```

---

## 最小完整示例

```cpp
#include <pthread.h>
#include <cstdio>

struct Task {
    int id;
    const char* name;
};

void* worker(void* data) {
    Task* task = static_cast<Task*>(data);
    std::printf("worker: %d %s\n", task->id, task->name);
    return nullptr;
}

int main() {
    pthread_t thread;
    Task task{7, "parse packet"};

    int result = pthread_create(&thread, nullptr, worker, &task);
    if (result != 0) {
        std::printf("pthread_create failed: %d\n", result);
        return 1;
    }

    std::printf("main thread\n");
    pthread_join(thread, nullptr);
}
```

Linux 下编译时要链接 pthread：

```bash
g++ main.cpp -pthread -o main
```

`main thread` 和 `worker` 的输出先后不能仅凭源码顺序确定，因为创建成功后两条执行流会并发推进。`pthread_join` 保证主线程在退出前等待目标线程结束。

---

## 参数生命周期是第一大坑

pthread 只保存并传递地址，不会自动复制 `arg` 指向的对象。

因此必须保证：

- 新线程使用参数期间，对象仍然存活
- 对象没有被提前释放
- 如果多个线程同时修改对象，必须另外进行同步

下面这种循环写法尤其危险：

```cpp
for (int id = 0; id < 4; ++id) {
    pthread_create(&threads[id], nullptr, worker, &id);
}
```

所有线程可能拿到同一个 `id` 地址，而且主线程还在不断修改它。这同时涉及参数值不稳定和数据竞争问题。

---

## 与 `std::thread` 对照

| POSIX pthread | C++ `std::thread` | 作用 |
|---|---|---|
| `pthread_t` | `std::thread` 对象 | 保存线程标识或管理句柄 |
| `pthread_create` | `std::thread(...)` | 创建线程 |
| `start_routine + void* arg` | 可调用对象 + 参数 | 指定入口和上下文 |
| `pthread_join` | `thread.join()` | 等待线程结束 |
| `pthread_detach` | `thread.detach()` | 分离线程 |
| 手动检查返回码 | 异常或对象状态 | 报告创建和管理错误 |

`std::thread` 提供了类型安全、模板参数保存和 RAII 风格的 C++ 接口；pthread 则暴露了更统一、更接近系统层的 C 风格 ABI。

---

## 当前最容易混淆的点

1. `pthread_attr_t` 是创建配置，不是线程本身。
2. `pthread_attr_init` 初始化已有对象，不是声明或分配这个对象。
3. `void` 和 `void*` 不是一回事。
4. `start_routine` 是函数指针参数，`arg` 是数据指针参数，它们不是两个函数。
5. `arg` 指向的对象必须活得足够久。
6. `pthread_create` 返回非零错误码时，线程没有创建成功。

---

## 下一步

1. `pthread_create` 的错误码处理
2. `pthread_join` 如何接收线程返回的 `void*`
3. `pthread_detach` 与 joinable 资源回收
4. `pthread_mutex_t` 与 `pthread_cond_t`

## 相关页面

- [[02-Linux/Linux 系统编程]]
- [[04-C-Cpp/C++ std thread 入门：线程入口、join 与参数传递]]
- [[04-C-Cpp/C++ 线程深入理解：执行流、共享内存与生命周期]]
- [[04-C-Cpp/C++ 线程同步入门：竞态条件、mutex 与 lock_guard]]
