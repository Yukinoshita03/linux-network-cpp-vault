# mini-bgp-or-ospf 项目总览

## 目标

实现一个简化控制面协议，体现 C++ 状态机、事件驱动、多线程和路由表管理能力。

## 推荐选择

优先做 mini-BGP。原因：

- 运行在 TCP 之上，调试成本低于直接处理 OSPF IP protocol 89。
- 状态机清晰。
- 和 FRR 对接更直观。
- 面试容易讲出协议理解。

## mini-BGP 功能

1. TCP 连接管理。
2. BGP OPEN 报文。
3. KEEPALIVE。
4. 简化 UPDATE。
5. Peer 状态机。
6. 路由表管理。
7. 定时器。
8. 和 FRR 建邻测试。

## C++ 设计点

- `Peer`
- `Session`
- `MessageCodec`
- `RouteTable`
- `TimerManager`
- `EventLoop`
- `StateMachine`

## 简历描述

使用 C++ 实现简化 BGP 控制面，支持 Peer 状态机、OPEN/KEEPALIVE/UPDATE 报文编解码、路由表维护和定时器管理。采用事件驱动结构处理 TCP 会话，并通过 FRR 进行邻居建立和路由通告验证。
