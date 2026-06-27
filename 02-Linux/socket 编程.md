# socket 编程

## 需要掌握

- `socket`
- `bind`
- `listen`
- `accept`
- `connect`
- `send`
- `recv`
- `sendto`
- `recvfrom`
- `setsockopt`

## 常见 socket 类型

- TCP：`SOCK_STREAM`
- UDP：`SOCK_DGRAM`
- 原始套接字：`SOCK_RAW`

## 面试回答

TCP socket 面向连接，内核维护连接状态、重传、拥塞控制和流量控制。UDP socket 面向报文，不保证可靠性和顺序。raw socket 可以绕过部分传输层封装，直接处理 IP 或链路层报文，适合做抓包、发包、简化路由器和协议实验。

## 练习

- 写 TCP echo server。
- 写 UDP echo server。
- 写 raw socket 打印 Ethernet、IPv4、ICMP 头部。
