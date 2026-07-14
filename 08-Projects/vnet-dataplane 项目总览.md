# vnet-dataplane 项目总览

## 项目定位

`vnet-dataplane` 是一个面向 C++ 基础设施和 Linux 网络方向求职的项目，目标是实现：

- 一个 Linux 虚拟网卡驱动
- 一个 C++20 用户态 dataplane
- 一套围绕报文解析、过滤、转发和观测的工程骨架

对应仓库：`https://github.com/Yukinoshita03/vnet-dataplane`

## 为什么选这个题

- 比普通 WebServer 更偏底层，能展示 Linux 网络栈理解
- 比真实硬件网卡驱动风险更低，更适合个人项目落地
- 同时覆盖 C++、网络协议、内核接口、性能分析和工程化

## 当前阶段

第一阶段先不碰复杂驱动逻辑，先把 userspace parser 做稳：

1. 解析 Ethernet 头
2. 解析 IPv4 头
3. 解析 TCP/UDP 头
4. 补测试和一个最小 CLI

这样做的原因是，后续无论数据来自驱动 ring、抓包文件还是用户态 buffer，最终都需要先把原始字节流解析成结构化协议字段。

## 当前学习抓手

这个项目当前最适合拿来练：

- `struct` 和轻量数据模型设计
- `std::array`
- `std::optional`
- `std::span`
- 边界检查和字节序处理
- 用测试保护 parser 行为

## 关联

- [[03-Networking/TCP-IP 基础]]
- [[03-Networking/IPv4]]
- [[03-Networking/Ethernet]]
