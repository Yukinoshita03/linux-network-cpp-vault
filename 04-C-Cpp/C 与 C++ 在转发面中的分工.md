# C 与 C++ 在转发面中的分工

## 常见分工

- C：数据面、包处理、驱动、内核、DPDK、XDP。
- C++：控制面、管理面、协议状态机、配置、服务框架。

## 为什么转发面偏 C

转发面强调可控内存布局、低开销、可预测性能、少抽象成本。C 更接近内核和硬件接口。

## 为什么控制面偏 C++

控制面需要表达复杂状态、对象关系、定时器、事件通知、策略和配置管理。C++ 的 RAII、类、STL 和多态更适合组织大型协议代码。

## 必学 C++ 特性

- RAII
- 智能指针
- `std::vector`
- `std::map`
- `std::unordered_map`
- `std::thread`
- `std::mutex`
- `std::condition_variable`
- 状态机建模

## 不优先深挖

- 模板元编程
- SFINAE
- concepts
- 协程
- GUI 框架
