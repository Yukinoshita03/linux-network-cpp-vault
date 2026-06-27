# Ethernet

## 头部

```text
dst mac: 6 bytes
src mac: 6 bytes
ether type: 2 bytes
payload
```

常见 EtherType：

- `0x0800`：IPv4
- `0x0806`：ARP
- `0x86DD`：IPv6

## 转发时要改什么

三层转发时，IP 目的地址通常不变，但二层 MAC 会变化：

- 源 MAC 改为出接口 MAC。
- 目的 MAC 改为下一跳 MAC。

## 面试回答

二层交换根据目的 MAC 查 MAC 表转发，三层路由根据目的 IP 查路由表转发。三层转发每经过一跳都会重写二层头，同时 IPv4 TTL 减 1 并重算校验和。
