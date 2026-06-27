# Netfilter

## 五个 IPv4 hook

- `NF_INET_PRE_ROUTING`
- `NF_INET_LOCAL_IN`
- `NF_INET_FORWARD`
- `NF_INET_LOCAL_OUT`
- `NF_INET_POST_ROUTING`

## 常见用途

- 防火墙
- NAT
- 包过滤
- 连接跟踪
- 报文修改

## iptables/nftables

iptables 和 nftables 是用户态配置工具，底层能力来自内核 Netfilter。

## 面试回答

Netfilter 在 Linux 网络路径中提供多个 hook 点，内核模块可以在这些点检查、修改、丢弃或接受报文。iptables/nftables 只是配置规则的用户态入口。
