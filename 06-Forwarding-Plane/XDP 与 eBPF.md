# XDP 与 eBPF

## XDP 位置

XDP 在报文进入内核网络栈之前运行，位置靠近驱动层。

```text
NIC -> driver -> XDP -> Linux network stack
```

## 常见动作

- `XDP_PASS`：继续进入协议栈。
- `XDP_DROP`：直接丢弃。
- `XDP_TX`：从原网卡发回。
- `XDP_REDIRECT`：重定向到其他网卡或 map。

## 为什么快

- 更早处理包。
- 避免进入完整协议栈。
- 可以减少 `sk_buff` 分配。
- 适合 DDoS 过滤、快速丢包、简单转发。

## 面试回答

XDP 是基于 eBPF 的高速包处理机制，运行在驱动接收路径早期。相比 iptables，它可以在报文进入完整内核协议栈前完成过滤或转发，因此开销更低。
