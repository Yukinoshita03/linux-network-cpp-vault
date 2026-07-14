# Linux 收发包内存路径与 XDP DNS 快路径结晶化

日期：2026-07-12
来源：对话/工作会话
置信度：INFERRED

## 核心洞见

1. Linux 网络包每经过一层协议并不会复制一次。Ethernet、bridge、netfilter、IPv4、UDP 等模块通常围绕同一个 `sk_buff` 及其引用的底层 packet buffer 工作，主要成本是对象管理、查表、队列、规则执行、引用计数、CPU cache miss 和调度，而不一定是 payload `memcpy`。
2. 普通 UDP socket 路径中最明确的两次 CPU payload 拷贝，是 `recvfrom()` 将内核 skb 数据复制到用户缓冲区，以及 `sendto()` 将用户响应复制回内核发送 skb。网卡接收和发送则通常通过 DMA 在网卡与主机内存之间移动数据，不是 CPU 执行的 `memcpy`。
3. Native XDP 在完整 `sk_buff` 和通用协议栈之前处理驱动提供的 RX buffer。DNS 缓存命中时，它查询 BPF map、原地修改 MAC/IP/UDP/DNS 字段并返回 `XDP_TX`，避免 socket 队列、用户态进程唤醒、上下文切换、两次用户态边界复制和后端 DNS 服务访问。
4. Linux bridge 是二层软件交换机。单播已知目标端口时通常只转交一个 skb；广播需要向多个端口泛洪，Linux 常通过 `skb_clone()` 复制较小的 skb 元数据并共享底层 payload。只有某个分支需要修改共享报文时，才可能触发 copy-on-write 或数据复制。
5. XDP 快路径不是完整 DNS 服务的替代品。当前项目只加速 IPv4、UDP、单问题、未压缩 QNAME、`A/IN` 且缓存命中的请求；未命中、过期或不支持的请求返回 `XDP_PASS`，继续交给完整协议栈和正常 DNS 服务处理。

## 普通 UDP DNS 收包与回包路径

```text
网卡接收帧
  -> DMA 写入驱动预先准备的 RX buffer
  -> NAPI 批量轮询 RX descriptor
  -> 构造 skb 元数据并引用 packet buffer
  -> Ethernet / bridge / netfilter
  -> IPv4、路由、iptables/nftables
  -> UDP socket 查找
  -> skb 进入 socket 接收队列
  -> 唤醒 DNS 服务进程
  -> recvfrom(): kernel -> userspace payload copy
  -> DNS 服务解析并查询用户态缓存
  -> 用户态构造响应
  -> sendto(): userspace -> kernel payload copy
  -> UDP/IP、路由、netfilter、qdisc
  -> 网卡 DMA 读取内核发送 buffer
  -> 发送到网络
```

各协议层大多通过 `skb->mac_header`、`skb->network_header`、`skb->transport_header` 等偏移解释同一份报文数据，而不是把 Ethernet、IP、UDP 头分别复制到新内存。

## Bridge 单播与广播的内存语义

典型物理/虚拟混合拓扑：

```text
外部网络 <-> ens33/eth0 <-> br0 <-> veth/tap <-> 容器或虚拟机
```

- `ens33/eth0`：连接外部网络的接口；在 VMware guest 中它仍可能是虚拟网卡，但对 guest Linux 来说承担外部网卡角色。
- `veth`：成对的虚拟链路，常用于连接 network namespace 或容器。
- `tap`：常用于连接虚拟机网卡。
- `br0`：把这些端口接入同一个二层广播域。

已知目标 MAC 的单播：

```text
eth0 -> br0 -> veth0
```

通常可以把原 skb 交给唯一输出端口，不需要复制完整 payload。

广播：

```text
eth0 -> br0 -> veth0
           -> veth1
           -> tap0
```

多个输出路径需要各自的 skb 元数据，但可以共享同一块 packet data：

```text
skb_original ---+
skb_clone_1 ----+--> shared packet data (refcount)
skb_clone_2 ----+
```

`skb_clone()` 通常是浅克隆。若某个输出分支必须修改共享数据，才需要先解除共享或复制后再修改。

## XDP DNS 缓存命中路径

```text
网卡 DMA RX
  -> Native XDP 读取 RX buffer
  -> 解析 Ethernet / IPv4 / UDP / DNS
  -> 用 QNAME + QTYPE + QCLASS 查询 dns_cache BPF map
  -> 检查 TTL/过期时间
  -> bpf_xdp_adjust_tail() 增加 DNS A answer 空间
  -> 交换源/目的 MAC、IP、UDP 端口
  -> 写 DNS response flags、answer、长度和校验和
  -> XDP_TX
  -> 网卡 DMA TX
```

与普通路径相比，命中时没有进入：

```text
完整 skb 协议栈
socket 查找和接收队列
DNS 用户态进程
recvfrom()/sendto() 用户态边界复制
后端 DNS 服务
普通 UDP/IP 输出路径
```

因此答辩时应准确表述为：XDP 的性能收益来自“不进入用户态”和“在完整通用协议栈之前结束处理”两部分叠加，而不只是省掉一次函数调用。

## Native、Generic XDP 与 tc

```text
Native XDP  -> 驱动收包早期，通常在完整 skb 之前，性能最好但要求驱动支持
Generic XDP -> skb 路径上的兼容实现，部署容易但收益低于 Native XDP
tc eBPF     -> 更靠后，已有 skb 和更多上下文，适合双向监控、分类和策略
用户态服务 -> 功能最完整，但路径最长
```

项目在 VMware/veth 场景使用 Generic XDP 便于复现；最终性能结论需要区分 Generic 与 Native XDP，不能把两者混为一谈。

## 关键决策

- 优先理解和验证 DNS XDP cache 这条最完整的加速闭环，再扩展 gRPC、OpenStack 和 Kubernetes 路径。
- 保留普通 DNS 服务作为兜底，只让明确安全且缓存命中的请求进入 XDP 快路径。
- 答辩时不将性能提升表述为“所有 DNS 请求都加速”，而限定为“当前支持范围内的缓存命中请求”。
- 分析性能时同时考虑数据复制、对象分配、队列、协议层、查表、上下文切换和 CPU cache，而不是只统计 `memcpy` 次数。

## 涉及概念

- [[06-Forwarding-Plane/XDP 与 eBPF]]
- [[05-Kernel-Networking/Linux 内核网络栈地图]]
- [[03-Networking/TCP-IP 基础]]
- [[08-Projects/vnet-dataplane 项目总览]]

## 参考资料

- `vnet-dataplane/linux_accel/bpf/dns_xdp_monitor.c`
- `vnet-dataplane/linux_accel/src/dns_monitor.cpp`
- `vnet-dataplane/linux_accel/src/dns_cache_config.cpp`
- 当前项目学习与代码讲解对话

## 待跟进

- 对照源码逐行理解 `dns_xdp_monitor()` 与 `try_dns_cache_response()`。
- 继续学习用户态 `dns_cache_key` 编码与 BPF map 查找为什么必须拥有完全一致的内存布局。
- 在 Linux VM 中分别测量 no-hook、tc monitor、Generic XDP cache hit，并记录 QPS 与 p99。
- 后续在支持 Native XDP 的网卡和驱动上复测，区分 Generic XDP 与 Native XDP 的真实性能收益。
