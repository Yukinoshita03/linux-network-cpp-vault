# tcpdump

## 常用命令

```bash
tcpdump -i any -nn
tcpdump -i eth0 -nn icmp
tcpdump -i eth0 -nn arp
tcpdump -i eth0 -e -nn
tcpdump -i eth0 -w out.pcap
```

## 参数

- `-i`：指定网卡。
- `-nn`：不解析主机名和端口名。
- `-e`：显示二层头。
- `-w`：写入 pcap 文件。

## 面试价值

做网络项目时，要用抓包证明自己的判断。比如 ARP 是否发出、ICMP Echo Request 是否到达路由器、转发后的源/目的 MAC 是否正确。
