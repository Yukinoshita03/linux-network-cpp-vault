# TCP-IP 基础

## 分层

```text
应用层：HTTP、DNS、BGP
传输层：TCP、UDP
网络层：IPv4、IPv6、ICMP
链路层：Ethernet、ARP、VLAN
物理层：网线、光纤、无线
```

## 转发面最重要的问题

收到一个包后：

1. 目的 MAC 是不是本机？
2. EtherType 是什么？
3. 目的 IP 是不是本机？
4. 如果不是本机，能不能转发？
5. 路由表查到哪个下一跳？
6. 下一跳 MAC 是否已知？
7. TTL、校验和、二层头如何更新？

## 关联

- [[03-Networking/Ethernet]]
- [[03-Networking/ARP]]
- [[03-Networking/IPv4]]
- [[03-Networking/ICMP]]
