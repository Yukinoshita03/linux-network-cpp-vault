# BGP

## 核心概念

- Peer
- AS
- OPEN
- KEEPALIVE
- UPDATE
- NOTIFICATION
- Path attributes

## 状态机

```text
Idle -> Connect -> Active -> OpenSent -> OpenConfirm -> Established
```

## 项目简化版

可以先只做：

- TCP 179 连接
- OPEN 协商
- KEEPALIVE 保活
- UPDATE 通告简单前缀
- 路由表更新

## 面试回答

BGP 是路径向量协议，运行在 TCP 之上。BGP 通过 UPDATE 报文传播可达前缀和路径属性，使用策略选择最优路由。它更关注自治系统之间的可达性和策略，而不是单纯最短路径。
