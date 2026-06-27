# epoll

## 作用

`epoll` 用于在 Linux 上高效监听大量文件描述符事件。

## 三个核心调用

```c
epoll_create1
epoll_ctl
epoll_wait
```

## LT 与 ET

- LT：水平触发，只要 fd 还有数据就持续通知。
- ET：边缘触发，状态变化时通知，通常需要非阻塞 fd，并一次读到 `EAGAIN`。

## 面试回答

`epoll` 的价值是避免对大量 fd 做线性轮询。程序把关心的 fd 注册进内核，事件到来后由 `epoll_wait` 返回就绪列表。网络服务常用 `epoll + nonblocking socket`，避免一个连接阻塞整个事件循环。

## 项目落点

在控制面协议项目中，可以用 `epoll` 处理多个邻居连接和定时器事件。
