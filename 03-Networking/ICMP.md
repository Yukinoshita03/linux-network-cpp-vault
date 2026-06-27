# ICMP

## 常见类型

- Echo Request
- Echo Reply
- Destination Unreachable
- Time Exceeded

## mini-router 中的作用

- ping 路由器接口 IP：路由器应回复 Echo Reply。
- ping 远端主机：路由器应转发 ICMP 报文。
- TTL 过期：路由器应回复 Time Exceeded。

## 面试回答

ICMP 是网络层控制报文，用于错误报告和诊断。`ping` 使用 ICMP Echo Request/Reply，`traceroute` 利用 TTL 逐跳过期触发 ICMP Time Exceeded。
