# ip_rcv 到 ip_forward

## 路径

```text
ip_rcv
 -> NF_INET_PRE_ROUTING
 -> ip_rcv_finish
 -> ip_route_input_noref
 -> dst_input
 -> ip_forward
 -> NF_INET_FORWARD
 -> ip_forward_finish
 -> dst_output
 -> ip_output
 -> NF_INET_POST_ROUTING
 -> dev_queue_xmit
```

## 转发关键点

- 路由查找决定本机接收还是转发。
- Netfilter 在多个 hook 点可以修改、丢弃或放行包。
- 转发时 TTL 减 1。
- 出口阶段进入 qdisc，然后到驱动发送。

## 面试回答

IPv4 包进入内核后先经过 `ip_rcv`，再走 `PRE_ROUTING` hook，随后做路由查找。如果目的地址属于本机，进入本地协议栈；如果需要转发，则进入 `ip_forward`，经过 `FORWARD` 和 `POST_ROUTING` hook，最后从出接口发送。
