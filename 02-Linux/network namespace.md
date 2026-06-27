# network namespace

## 用途

network namespace 可以在一台 Linux 机器上模拟多台独立主机。每个 namespace 有自己的网卡、IP 地址、路由表和防火墙规则。

## 常用命令

```bash
ip netns add h1
ip netns add r1
ip netns exec h1 ip addr
ip link add veth-h1 type veth peer name veth-r1
ip link set veth-h1 netns h1
ip link set veth-r1 netns r1
```

## mini-router 拓扑

```text
h1 ---- r1 ---- h2
```

`r1` 上运行用户态路由器，负责转发 h1 和 h2 之间的 ICMP 报文。

## 验证方式

```bash
ip netns exec h1 ping 10.0.2.2
ip netns exec r1 tcpdump -i any -nn
```
