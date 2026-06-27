# mini-router 项目总览

## 目标

实现一个用户态简化路由器，在 Linux network namespace 中完成 ICMP 跨网段转发。

## 拓扑

```text
h1: 10.0.1.2/24
  |
r1-eth0: 10.0.1.1/24
r1-eth1: 10.0.2.1/24
  |
h2: 10.0.2.2/24
```

## 功能拆分

1. 搭建 namespace 拓扑。
2. raw socket 收包。
3. 解析 Ethernet。
4. 解析 ARP。
5. 解析 IPv4。
6. 支持 ICMP Echo Reply。
7. 实现 ARP 表。
8. 实现路由表。
9. 实现 LPM。
10. 转发 IPv4 包。
11. TTL 和 checksum 处理。
12. tcpdump 验证。

## 项目亮点

- 真实 Linux 虚拟网络环境。
- 报文级处理。
- 路由表和 ARP 表。
- LPM 最长前缀匹配。
- 抓包验证转发路径。

## 简历描述

实现基于 raw socket 的用户态简化路由器，支持 Ethernet/ARP/IPv4/ICMP 报文解析、ARP 表维护、路由表最长前缀匹配和跨网段 ICMP 转发。使用 Linux network namespace 构建三节点拓扑，并通过 tcpdump 验证 ARP 解析、TTL 更新、checksum 重算和二层头重写流程。
