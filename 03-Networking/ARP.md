# ARP

## 作用

ARP 用于在 IPv4 局域网中根据 IP 地址获取 MAC 地址。

## 两类报文

- ARP Request：广播询问某个 IP 的 MAC。
- ARP Reply：单播回复自己的 MAC。

## mini-router 中的作用

路由器查到下一跳 IP 后，如果不知道下一跳 MAC，就需要先发 ARP Request。收到 ARP Reply 后，再把等待中的 IP 包发出去。

## 面试回答

ARP 是 IPv4 到 Ethernet 之间的地址解析机制。路由器转发 IP 包时，路由表决定下一跳 IP，ARP 表决定下一跳 MAC。没有 ARP 结果时，不能直接构造正确的二层头。
