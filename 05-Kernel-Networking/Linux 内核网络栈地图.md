		# Linux 内核网络栈地图

## 关键结构

- [[05-Kernel-Networking/sk_buff]]
- `net_device`
- `napi_struct`
- `dst_entry`
- FIB 路由表

## IPv4 收包大致路径

```text
NIC
 -> driver
 -> NAPI poll
 -> sk_buff
 -> __netif_receive_skb_core
 -> ip_rcv
 -> ip_rcv_finish
 -> 路由查找
 -> 本机接收 or 转发
 -> ip_forward
 -> ip_output
 -> dev_queue_xmit
 -> driver
 -> NIC
```

## 需要读的源码位置

```text
net/core/dev.c
net/ipv4/ip_input.c
net/ipv4/ip_forward.c
net/ipv4/route.c
net/ipv4/fib_trie.c
net/netfilter/
```

## 学习方法

先用图和调用链理解路径，再读源码。不要从第一行开始啃。
