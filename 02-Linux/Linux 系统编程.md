# Linux 系统编程

## 核心主题

- 文件描述符
- 阻塞与非阻塞 IO
- socket
- epoll
- 多线程
- 定时器
- 信号
- 进程间通信

## 为什么重要

协议栈和转发面程序通常是长期运行的后台服务。它们需要稳定地处理大量连接、定时器、报文和状态变化。

## 学习顺序

1. [[02-Linux/socket 编程]]
2. [[02-Linux/epoll]]
3. [[02-Linux/network namespace]]
4. [[02-Linux/tcpdump]]

## 项目落点

- epoll echo server
- raw socket packet sniffer
- mini-router 收发包主循环
