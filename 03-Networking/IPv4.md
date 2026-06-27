# IPv4

## 关键字段

- version
- ihl
- total length
- ttl
- protocol
- header checksum
- source address
- destination address

## 转发时要处理

1. 检查头部合法性。
2. 判断目的 IP 是否本机。
3. TTL 减 1。
4. TTL 为 0 时发送 ICMP Time Exceeded。
5. 查路由表。
6. 重算 IPv4 header checksum。
7. 重写二层头并发出。

## 面试回答

IPv4 转发的核心是目的地址查表。每经过一个三层设备，TTL 减 1，头部校验和需要更新。三层地址保持端到端不变，二层地址逐跳变化。
