# Ethernet

## L2 在做什么

Ethernet 是最常见的二层协议。L2 解决的问题不是跨网段路由，而是同一个二层网络里一个帧该发给谁、交换机该怎么转发。

可以先记住这三句话：

- L2 看 MAC 地址。
- L3 看 IP 地址。
- L4 看端口。

## 以太网头长什么样

标准 Ethernet II 头通常是 14 字节：

```text
| dst MAC 6B | src MAC 6B | EtherType 2B | payload ... |
```

其中：

- `dst MAC`：目的网卡地址
- `src MAC`：源网卡地址
- `EtherType`：上层协议类型

常见 EtherType：

- `0x0800`：IPv4
- `0x0806`：ARP
- `0x86DD`：IPv6
- `0x8100`：802.1Q VLAN

## MAC 地址

MAC 地址长度是 48 bit，也就是 6 字节。

常见类型：

- 单播：发给一个网卡
- 广播：`ff:ff:ff:ff:ff:ff`
- 组播：发给一组接收者

二层转发时，真正决定“这个帧往哪走”的是目的 MAC，不是目的 IP。

## 交换机在二层做什么

交换机核心做两件事：

- 学习：记住某个 MAC 出现在哪个端口后面
- 转发：看到目的 MAC 后，把帧发到对应端口

如果交换机还不知道目的 MAC 在哪，它会在同一个 VLAN 里泛洪这个帧。

## VLAN 是做什么的

VLAN 是二层隔离手段，用来把一个物理交换网络切成多个逻辑二层网络。它最常见的作用是：

- 隔离不同业务或租户流量
- 控制广播域大小
- 在不改物理布线的前提下划分多个逻辑网络

带 802.1Q VLAN 的以太帧会在 Ethernet 头和上层协议之间多插入一个 4 字节 tag：

```text
| dst MAC 6B | src MAC 6B | 0x8100 2B | VLAN tag 4B | inner EtherType 2B | payload ... |
```

所以：

- 不带 VLAN 的以太头通常是 14 字节
- 带单层 802.1Q VLAN 时通常是 18 字节

## ARP 和 Ethernet 的关系

IPv4 包最终还是要封装进 Ethernet 帧发出去，所以只知道目标 IP 还不够，还要知道下一跳 MAC。ARP 就是用来做这件事的。

可以这样理解：

- 路由表决定“下一跳 IP 是谁”
- ARP 表决定“下一跳 MAC 是谁”

没有正确的目的 MAC，就构不出正确的二层头。

## 三层转发时二层头会改什么

三层转发时，IP 目的地址通常不变，但二层 MAC 会变化：

- 源 MAC 改为出接口 MAC
- 目的 MAC 改为下一跳 MAC

同时 IPv4 转发通常还会：

- `TTL - 1`
- 重算 IPv4 头校验和

这也是为什么“IP 是端到端地址，MAC 是逐跳地址”是一个很重要的基础概念。

## 从 XDP/eBPF 视角看 L2

在 XDP 里，最常见的二层工作不是“讲协议历史”，而是：

- 先解析 `struct ethhdr`
- 检查 `EtherType`
- 如果是 `0x8100`，继续处理 VLAN tag
- 必要时改写 `h_dest` 和 `h_source`
- 把包 redirect 到正确接口

对应到你在 XDP 项目里看到的代码，`parse_ethhdr()` 做的就是这一步：先读 Ethernet 头，必要时再跳过单层 VLAN，给后面的 IP 解析准备好正确的偏移。

## 面试回答

二层交换根据目的 MAC 查 MAC 表转发，三层路由根据目的 IP 查路由表转发。三层转发每经过一跳，都会重写二层头；如果是 IPv4，还会让 TTL 减 1 并更新校验和。VLAN 的作用是在二层做逻辑隔离，ARP 的作用是在 IPv4 和 Ethernet 之间完成下一跳 MAC 解析。

## 关联

- [[03-Networking/ARP]]
- [[03-Networking/IPv4]]
- [[03-Networking/TCP-IP 基础]]
- [[06-Forwarding-Plane/XDP 与 eBPF]]
