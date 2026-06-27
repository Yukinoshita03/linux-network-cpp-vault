# OSPF

## 核心概念

- Hello
- Neighbor
- LSA
- LSDB
- SPF
- Area

## 邻居状态

常见状态：

```text
Down -> Init -> 2-Way -> ExStart -> Exchange -> Loading -> Full
```

## 项目简化版

可以先只做：

- Hello 报文
- 邻居发现
- 简化 LSDB
- Dijkstra 计算路由

## 面试回答

OSPF 是链路状态路由协议。每台路由器通过 LSA 同步拓扑信息，形成 LSDB，然后运行 SPF 算法计算最短路径，最终生成路由表。
