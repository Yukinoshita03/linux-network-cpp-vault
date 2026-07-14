# TCP-IP 基础

## 分层

```text
应用层：HTTP、DNS、BGP
传输层：TCP、UDP
网络层：IPv4、IPv6、ICMP
链路层：Ethernet、ARP、VLAN
物理层：网线、光纤、无线
```

可以把常见字段也顺手挂到这一层分工上：

- L2 看 MAC 地址，决定同一个二层网络里的帧该交给谁
- L3 看 IP 地址，决定包该去哪台主机
- L4 看端口号，决定交给这台主机上的哪个进程

对应到 parser 里，最常见的解析链路就是：

```text
Ethernet 头 -> IPv4 头 -> TCP/UDP 头 -> payload
```

也就是说：

- 先看 `EtherType`，判断上层是不是 IPv4
- 再看 IPv4 的 `protocol` 字段，判断后面跟的是 TCP 还是 UDP
- 最后才按 TCP/UDP 头去读端口、长度、控制位等字段

## 转发面最重要的问题

收到一个包后：

1. 目的 MAC 是不是本机？
2. EtherType 是什么？
3. 目的 IP 是不是本机？
4. 如果不是本机，能不能转发？
5. 路由表查到哪个下一跳？
6. 下一跳 MAC 是否已知？
7. TTL、校验和、二层头如何更新？

## 从原始字节到协议字段

做用户态 dataplane 或抓包分析时，拿到的通常不是高级对象，而是一段原始字节流。第一步往往不是“直接处理业务”，而是分层解析：

1. 解析 Ethernet 头，拿到源 MAC、目的 MAC、`EtherType`
2. 如果 `EtherType = 0x0800`，继续解析 IPv4 头
3. 读取 IPv4 `protocol`
4. `protocol = 6` 时按 TCP 头解析
5. `protocol = 17` 时按 UDP 头解析

这也是后续做 ACL、五元组过滤、简单转发和 metrics 的基础。

## 关联

- [[03-Networking/Ethernet]]
- [[03-Networking/ARP]]
- [[03-Networking/IPv4]]
- [[03-Networking/ICMP]]
