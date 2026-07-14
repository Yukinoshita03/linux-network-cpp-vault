# `C++ 生产者消费者模型` 入门：`mutex`、`condition_variable` 与队列协作

## 为什么这一页重要

如果你刚学完：

- `std::thread`
- `mutex`
- `unique_lock`
- `condition_variable`

那最自然的下一步不是继续背新 API，
而是把它们真正拼成一个可运行模型。

生产者消费者就是这条线上最标准的落地：

- 生产者负责放数据
- 消费者负责取数据
- 队列空时消费者要等
- 队列有数据后再被唤醒

这正好把前面学的同步原语全部串起来。

---

## 先看它到底在解决什么问题

假设有一个共享队列：

- 生产者往里 `push`
- 消费者从里 `pop`

最核心的问题不是“怎么操作队列”，
而是：

- **队列为空时，消费者怎么办**

它不能：

1. 一直 `while (q.empty()) {}` 空转
2. 一边拿着锁一边死等

因为：

- 空转会浪费 CPU
- 持锁死等会把生产者堵在外面

所以正确思路是：

- 队列空了，消费者先睡
- 睡前把锁让出来
- 生产者放入数据后发通知
- 消费者醒来后重新拿锁，再检查队列

这就是生产者消费者模型的骨架。

---

## 这个模型里每个组件各干什么

### 1. 共享队列

它是共享状态。

比如：

```cpp
std::queue<int> q;
```

### 2. `mutex`

它负责保护队列本身，
避免多个线程同时改它。

### 3. `condition_variable`

它不保护队列，
它只负责：

- 队列空时让消费者睡
- 队列非空时把消费者叫醒

### 4. 条件

最常见的条件就是：

```cpp
!q.empty()
```

也就是：

- 队列里有数据了，消费者才能继续

---

## 最小代码骨架

```cpp
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <queue>
#include <thread>

std::mutex m;
std::condition_variable cv;
std::queue<int> q;

void producer() {
    for (int i = 1; i <= 5; ++i) {
        {
            std::lock_guard<std::mutex> lock(m);
            q.push(i);
        }
        cv.notify_one();
    }
}

void consumer() {
    for (int i = 1; i <= 5; ++i) {
        std::unique_lock<std::mutex> lock(m);
        cv.wait(lock, [] { return !q.empty(); });

        int value = q.front();
        q.pop();
        lock.unlock();

        std::cout << "consume: " << value << "\n";
    }
}
```

这段代码你先别急着背，
先看清它的执行顺序。

---

## 消费者这边到底发生了什么

看这几行：

```cpp
std::unique_lock<std::mutex> lock(m);
cv.wait(lock, [] { return !q.empty(); });
```

它的意思不是“在这里卡住”这么简单，
而是：

1. 先拿锁
2. 检查 `!q.empty()`
3. 如果队列空，就放锁睡觉
4. 生产者放入数据后发通知
5. 消费者被唤醒
6. 重新拿锁
7. 再检查 `!q.empty()`
8. 条件成立才继续往下走

所以消费者真正等的不是“通知”，
而是：

- **队列真的非空**

---

## 生产者这边到底发生了什么

看这几行：

```cpp
{
    std::lock_guard<std::mutex> lock(m);
    q.push(i);
}
cv.notify_one();
```

它的顺序是：

1. 先拿锁
2. 修改共享队列
3. 释放锁
4. 发通知

这背后的工程直觉是：

- **先把状态改对，再叫别人来看**

也就是说，
`notify_one()` 不是在“创造条件”，
而是在说：

- “我已经把条件改好了，你可以起来重新检查了”

---

## 为什么消费者一定要用 `wait(lock, pred)`

因为消费者不能只写：

```cpp
cv.wait(lock);
```

然后一醒来就直接 `pop()`。

原因有两个：

### 1. 可能假唤醒

线程可能醒了，
但队列不一定真的非空。

### 2. 可能被别的消费者抢先拿走数据

如果有多个消费者，
你虽然被唤醒了，
但另一个消费者可能先一步拿走了队列里的元素。

所以安全写法必须是：

```cpp
cv.wait(lock, [] { return !q.empty(); });
```

或者：

```cpp
while (q.empty()) {
    cv.wait(lock);
}
```

---

## 为什么 `pop` 之后常常会尽快解锁

这几行很值得注意：

```cpp
int value = q.front();
q.pop();
lock.unlock();

std::cout << "consume: " << value << "\n";
```

这里的意思是：

- 共享队列相关操作做完了
- 后面的打印和队列无关
- 那就尽快把锁释放掉

这样做的好处是：

- 别的线程可以更早进入临界区
- 减少持锁时间
- 降低锁竞争

这就是一个很典型的并发习惯：

- **只在真正需要保护共享状态的那小段时间里持锁**

---

## 这个模型里最容易犯的错

### 1. 队列空时忙等

```cpp
while (q.empty()) {
}
```

这是空转烧 CPU。

### 2. 拿着锁死等

```cpp
std::unique_lock<std::mutex> lock(m);
while (q.empty()) {
}
```

这会把生产者挡住，
让队列永远没法变非空。

### 3. 只等通知，不等条件

```cpp
cv.wait(lock);
int value = q.front();
q.pop();
```

这会把“通知到了”和“条件成立了”混成一回事。

### 4. 修改共享状态不加锁

比如生产者直接：

```cpp
q.push(i);
cv.notify_one();
```

这样队列就暴露在竞态条件里了。

---

## 你现在最该记住的 4 句话

1. 队列是共享状态，所以必须用 `mutex` 保护。
2. `condition_variable` 不是保护队列，而是让“队列空时先睡”这件事成立。
3. 消费者等的不是通知本身，而是 `!q.empty()` 这个条件。
4. 生产者的典型顺序是：先改状态，再通知。

---

## 面试够用回答

如果别人问：

“生产者消费者模型在 C++ 里怎么写？”

你可以这样答：

生产者消费者模型通常会用一个共享队列保存任务或数据，用 `mutex` 保护队列本身，用 `condition_variable` 协调“队列为空时消费者等待、队列有数据时生产者唤醒消费者”。  
消费者常见写法是 `cv.wait(lock, [] { return !q.empty(); });`，这样能避免忙等，也能防止假唤醒或多消费者竞争下直接误消费。

---

## 一句话记忆

生产者消费者模型本质上就是：

- **用锁保护队列，用条件变量协调“没货先睡，有货再醒”。**

## 相关页面

- [[04-C-Cpp/C++ condition_variable 入门：等待、唤醒与生产者消费者]]
- [[04-C-Cpp/C++ 线程同步入门：竞态条件、mutex 与 lock_guard]]
- [[04-C-Cpp/C++ 线程入门总览：执行流、同步与学习顺序]]
- [[04-C-Cpp/C++ 容器与对象语义下一步学习路线]]
- [[04-C-Cpp/C++ 线程池入门：任务队列、工作线程与条件变量]]
