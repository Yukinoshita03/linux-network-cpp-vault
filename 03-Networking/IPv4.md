# IPv4

## 关键字段

- version
- ihl
- total length
- identification
- flags
- fragment offset
- ttl
- protocol
- header checksum
- source address
- destination address

## IPv4 头在分层里负责什么

IPv4 是网络层协议，核心职责是把包从一台主机送到另一台主机。

最值得先抓住的字段是：

- `source address` / `destination address`：源 IP 和目的 IP
- `protocol`：后面跟的是哪种传输层协议
- `ttl`：还能过几跳

`protocol` 常见取值：

- `6`：TCP
- `17`：UDP
- `1`：ICMP

所以 IPv4 头并不直接告诉你“交给哪个应用”，它只告诉你“交给哪种上层协议继续解析”。真正决定应用入口的是 TCP/UDP 头里的端口号。

## version 和 IHL 为什么共用一个字节

IPv4 头的第一个字节不是两个独立字段，而是：

```text
高 4 bit: version
低 4 bit: IHL
```

所以很多 parser 会把它先存成一个原始字节，再拆高低位，而不是直接写成两个独立字节字段。

- `version = 4` 表示 IPv4
- `IHL` 是 Internet Header Length，表示 IPv4 头有多长

`IHL` 的单位不是字节，而是 4 字节：

```text
IPv4 头长度 = IHL * 4
```

最常见的 `IHL = 5`，也就是 20 字节。这个字段的意义是告诉你 IPv4 头到哪里结束，后面的 TCP/UDP 头从哪里开始。

## 分片里的 fragment offset 是什么

IPv4 支持分片。一个较大的 IP 包拆成多片后，需要靠这些字段在接收端重组：

- `identification`：标识这些分片原本属于同一个 IP 包
- `flags`：尤其是 `MF`，表示后面是否还有分片
- `fragment offset`：当前分片在原始载荷中的相对位置

`fragment offset` 不是按字节计数，而是按 8 字节为单位：

```text
实际偏移字节数 = fragment_offset * 8
```

这样做的原因是字段位数有限，按 8 字节粒度能覆盖更大的偏移范围。

## UDP 伪头部和 IPv4 的关系

UDP 校验和在计算时，会临时借用一部分 IPv4 信息：

- 源 IP
- 目的 IP
- 协议号 `17`
- UDP 长度

这部分常被叫做“UDP 伪头部”。它只参与 checksum 计算，不会真的被发出去。

所以真实上网的内容仍然是：

```text
IPv4 头 + UDP 头 + payload
```

而不是：

```text
IPv4 头 + 伪头部 + UDP 头 + payload
```

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

## 关联

- [[03-Networking/TCP-IP 基础]]
- [[03-Networking/Ethernet]]
- [[08-Projects/vnet-dataplane 项目总览]]
