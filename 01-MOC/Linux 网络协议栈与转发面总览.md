# Linux 网络协议栈与转发面总览

## 目标岗位

目标方向：Linux 网络协议栈 / 通信协议栈 / 转发面 C/C++ 工程师。

关键词：

- Linux network stack
- C/C++
- socket / epoll
- raw socket
- routing / forwarding
- netfilter / nftables
- TC
- XDP / eBPF
- DPDK
- OSPF / BGP

## 能力地图

```text
应用层网络编程
  -> socket / epoll / 多线程
  -> raw socket / 抓包 / 发包
  -> 用户态路由器
  -> Linux 内核网络路径
  -> XDP / eBPF / TC
  -> DPDK / 高性能转发
  -> OSPF / BGP 控制面
```

## 四条主线

- [[03-Networking/TCP-IP 基础]]
- [[02-Linux/Linux 系统编程]]
- [[05-Kernel-Networking/Linux 内核网络栈地图]]
- [[06-Forwarding-Plane/转发面基础]]

## 两个作品

- [[08-Projects/mini-router 项目总览]]
- [[08-Projects/mini-bgp-or-ospf 项目总览]]

## 面试闭环

- 能讲清楚一个包从网卡进入 Linux 到被转发出去的路径。
- 能讲清楚最长前缀匹配 LPM 如何实现。
- 能讲清楚 DPDK/XDP 为什么快。
- 能讲清楚 OSPF/BGP 的核心状态机。
- 能拿出可以跑、可以抓包验证的项目。
