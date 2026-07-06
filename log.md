# 操作日志

## 2026-06-27

- 创建 Obsidian 知识库，主题为 Linux 网络协议栈与转发面 / C/C++。
- 迁移知识库到桌面目录。
- 初始化 Git 仓库并创建 GitHub 私有仓库 `Yukinoshita03/linux-network-cpp-vault`。
- 安装 GitHub CLI 并完成推送。
- 引入 llm-wiki 兼容结构：`.wiki-schema.md`、`index.md`、`purpose.md`、`wiki/`、`raw/`。

## 2026-07-03

- 扩充 `03-Networking/Ethernet.md`，补充 L2 常用知识，包括 MAC、交换机学习与泛洪、VLAN、ARP 与 Ethernet 的关系，以及 XDP/eBPF 视角下的二层解析与改写。
- 新增 `04-C-Cpp/STL 与常用容器入门.md`，把 STL 的核心概念、常用容器、选择原则、算法和常见坑沉淀到 Obsidian 主笔记。
- 新增 `04-C-Cpp/拷贝构造、移动构造与 vector 扩容.md`，沉淀对象语义与 `vector` 扩容成本之间的关系。
- 新增 `04-C-Cpp/noexcept 与 vector 为什么有时宁可拷贝不移动.md`，沉淀异常承诺与容器搬迁策略之间的关系。
- 新增 `04-C-Cpp/C++ 容器与对象语义下一步学习路线.md`，收束当前学习阶段并给出下一步顺序。
- 新增 `04-C-Cpp/左值、右值与 std move.md`，澄清“左值/右值不是等号左右边”的误区，并解释 `std::move` 的真实作用。
- 新增 `04-C-Cpp/T&&、引用折叠与完美转发.md`，继续补齐对象语义：右值引用、forwarding reference、引用折叠与 `std::forward`。
- 更新 `04-C-Cpp/T&&、引用折叠与完美转发.md`，补充转发语义的知识地图、学习层次与后续顺序。
- 新增 `04-C-Cpp/std forward、std move 与 emplace_back.md`，串起 `forward`、`move`、有名右值引用和 `emplace_back` 的真实落地。
- 更新 `04-C-Cpp/std forward、std move 与 emplace_back.md`，补充 `emplace_back` 的真实代码路径、`forward` 的必要性与和 `push_back` 的区别。
- 新增 `04-C-Cpp/vector 的迭代器、引用与指针失效.md`，沉淀 `vector` 修改操作下的失效规则与根因。
- 新增 `04-C-Cpp/vector 的 insert 与 erase 为什么贵.md`，沉淀连续内存约束下的中间插入/删除成本与遍历删除写法。
- 新增 `04-C-Cpp/vector 扩容、异常安全与 noexcept.md`，沉淀 `vector` 扩容时的真实流程、异常安全目标与 `noexcept` 对路径选择的影响。
- 新增 `04-C-Cpp/unordered_map 的底层、性能与适用场景.md`，沉淀哈希表结构、平均 O(1) 的前提、退化风险与和 `map` 的本质差别。

## 2026-07-04

- 更新 `04-C-Cpp/unordered_map 的底层、性能与适用场景.md`，补充 MSVC STL `xhash` 源码级笔记：`_Find_last` 如何通过 `hash & _Mask` 定位 bucket、为什么 bucket 数强制为 2 的幂、以及 `_Vec` 中 `lo/hi` 双槽位的组织方式。
- 更新 `04-C-Cpp/unordered_map 的底层、性能与适用场景.md`，补充“`_Hashval` 已经是 hash，为什么还要再 `& _Mask`”这一层区分：hash 值不等于 bucket 编号，后者只是前者在当前 bucket 数下的映射结果。
- 更新 `04-C-Cpp/unordered_map 的底层、性能与适用场景.md`，澄清 `_Vec` 的 `lo/hi` 双槽位不是讲解比喻，而是 MSVC `xhash` 里的字面实现，并补充 `_Insert_new_node_before()` 如何维护 bucket 边界。
- 更新 `04-C-Cpp/unordered_map 的底层、性能与适用场景.md`，补充 `_Find_last` 开头 4 行代码的运行时含义：`_Bucket`、`_Where`、`_End` 各自代表什么，以及为什么 `_Where == _End` 就等于“这个 bucket 为空”。
- 更新 `04-C-Cpp/unordered_map 的底层、性能与适用场景.md`，新增“整体结构图”章节，用 ASCII 图说明这版 MSVC `unordered_map` 里 `_List` 是元素存储、`_Vec` 是 bucket 边界索引，以及查找/插入时二者如何配合。
- 更新 `04-C-Cpp/unordered_map 的底层、性能与适用场景.md`，澄清讲解中的 `hi`/`lo` 与源码局部变量并不同名：`hi` 在 `_Find_last` 里是通过 `_Vec[(_Bucket << 1) + 1]` 直接读入 `_Where`，`lo` 才显式命名为 `_Bucket_lo`。
- 更新 `04-C-Cpp/unordered_map 的底层、性能与适用场景.md`，补充 bucket 的两个“index”在 MSVC `xhash` 里如何存储：`_Vec` 是压平的槽位数组 `[lo0, hi0, lo1, hi1, ...]`，每个槽里存的是指向 `_List` 节点的迭代器/指针，而不是整数下标。
- 更新 `04-C-Cpp/unordered_map 的底层、性能与适用场景.md`，补充 `_Insert_new_node_before()` 如何在插入后维护 `lo/hi`、为什么 `rehash` 重建的是 `_Vec` 而不是节点本体，以及和 `map/xtree` 红黑树模型在查找与插入路径上的根本差异。
- 新增 `04-C-Cpp/map 与红黑树底层实现.md`，从 MSVC STL `map/xtree` 源码切入，系统梳理 `map` 的红黑树模型：节点结构、head 哨兵结点、`_Find_lower_bound` 搜索路径、插入流程、旋转和平衡思路，以及和 `unordered_map` 的底层差异。
- 更新 `04-C-Cpp/C++ 容器与对象语义下一步学习路线.md`，新增“如果目标是暑期实习，当前深度是不是太深了”一节，把 STL 源码深挖和实习面试够用线拆开，并把后续重点收束到容器使用、对象语义、刷题、项目表达与 Linux/Git/调试基础。
- 更新 `04-C-Cpp/C++ 容器与对象语义下一步学习路线.md`，新增“暑期实习 2 周复习清单”，按 14 天拆分 C++ 容器/对象语义、算法高频题型、项目表达、Linux/并发基础与模拟面试的执行顺序。
- 更新 `04-C-Cpp/std forward、std move 与 emplace_back.md`，新增 `std::forward` / `std::move` 的面试回答模板，并补充它们“什么时候写了也可能无用”的高频追问：如 `move const 对象`、类型没有高效移动、轻量类型、返回局部对象乱写 `move`，以及 `forward` 用在非 forwarding reference 场景里并无真实转发价值。
- 更新 `04-C-Cpp/std forward、std move 与 emplace_back.md`，继续补充“误用”视角，区分“写了但无收益”和“语义写错”，并新增高频误用：对后续还要依赖原值的对象乱 `move`、在 forwarding reference 包装层里误把 `forward` 写成 `move`、把 `forward` 当成更高级的 `move`、以及对 moved-from 对象继续按原值语义依赖。
- 更新 `04-C-Cpp/std forward、std move 与 emplace_back.md`，补充一个高频概念纠偏：`std::forward` 不是“模板里声明右值时再用”，而是“模板 forwarding reference 场景里为了保留实参原始左/右值属性继续转发时用”，并与普通右值引用下更常见的 `std::move` 做对照。
- 更新 `04-C-Cpp/C++ 容器与对象语义下一步学习路线.md`，结合当前课表新增“对照你现在这张课表，接下来最该学什么”一节，明确建议停止继续深挖 STL 细节，先速过 C++ 新特性中的智能指针和 lambda，再切到 Linux 多线程、线程同步与 IPC。
- 更新 `04-C-Cpp/C++ 容器与对象语义下一步学习路线.md`，补充一个重要修正：如果迭代器和算法库还没系统学，则应先补齐 STL 的使用层闭环，优先学习 `begin/end`、迭代器能力差异、`[first, last)` 区间习惯，以及 `sort/find/count/lower_bound` 等高频算法，再进入智能指针与 Linux 多线程阶段。
- 新增 `04-C-Cpp/STL 迭代器入门：本质、区间与算法关系.md`，系统讲清迭代器的本质、为什么 STL 需要统一遍历接口、`begin/end` 与 `[first, last)` 区间习惯、不同容器迭代器能力差异，以及算法为什么围绕迭代器区间设计；同时在学习路线页补上这页作为下一步入口。

## 2026-07-06

- 新增 `04-C-Cpp/STL 常用算法入门：find、count、sort 与二分查找.md`，把 `find`、`count`、`reverse`、`sort`、`lower_bound`、`upper_bound`、`binary_search` 串成一组，重点讲清它们各自处理什么区间、返回什么结果、哪些是线性扫描、哪些依赖有序区间与迭代器能力。
- 更新 `04-C-Cpp/STL 迭代器入门：本质、区间与算法关系.md`，把新算法页挂到相关页面，和前一页的“下一步最自然接什么”形成闭环。
- 更新 `04-C-Cpp/C++ 容器与对象语义下一步学习路线.md`，把 STL 使用层的下一步入口明确补成“迭代器之后就接常用算法页”。
- 新增 `04-C-Cpp/STL 算法进一阶：比较器、max_element 与 remove-erase.md`，继续把算法库补到刷题常用层，集中讲 `max_element/min_element`、比较器、`stable_sort`、`remove` 不真删元素的本质，以及 `erase(remove(...), end())` 这一组高频惯用法。
- 更新 `04-C-Cpp/STL 常用算法入门：find、count、sort 与二分查找.md` 和 `04-C-Cpp/C++ 容器与对象语义下一步学习路线.md`，把算法库的下一跳明确接到“进一阶”这页，减少学习路线断点。
- 新增 `04-C-Cpp/C++ lambda 入门：本质、捕获与算法配合.md`，把 lambda 作为算法库之后的自然下一步，集中讲匿名函数对象的本质、值捕获与引用捕获、`mutable`、以及和 `sort/remove_if/find_if` 的典型配合方式。
- 新增 `04-C-Cpp/C++ 智能指针入门：unique_ptr、shared_ptr 与 weak_ptr.md`，把智能指针收束成“所有权模型”来讲，重点区分独占所有权、共享所有权、弱引用、循环引用与默认先用 `unique_ptr` 的选择原则。
- 更新 `04-C-Cpp/STL 算法进一阶：比较器、max_element 与 remove-erase.md` 以及 `04-C-Cpp/C++ 容器与对象语义下一步学习路线.md`，把学习路径继续接到 `lambda` 和智能指针，形成从算法库到现代 C++ 常用能力的过渡。
- 更新 `04-C-Cpp/C++ lambda 入门：本质、捕获与算法配合.md`，补充一个高频纠偏：`[base]` / `[&base]` 不是传参，而是捕获方式；前者更像拷贝副本，后者更像给外层变量起别名。
- 新增 `04-C-Cpp/C++ std function 入门：lambda、回调与统一可调用类型.md`，把 `std::function` 放到 lambda 之后来讲，重点解释它如何把普通函数、lambda、带捕获 lambda、函数对象统一成一个可存储、可传递的回调类型，并补充它和模板泛型在接口抽象与开销上的取舍。
- 更新 `04-C-Cpp/C++ lambda 入门：本质、捕获与算法配合.md`、`04-C-Cpp/C++ 智能指针入门：unique_ptr、shared_ptr 与 weak_ptr.md` 和 `04-C-Cpp/C++ 容器与对象语义下一步学习路线.md`，把学习路径继续接到 `std::function`，为后续 `std::thread` 与并发任务模型做过渡。
- 更新 `04-C-Cpp/C++ std function 入门：lambda、回调与统一可调用类型.md`，补充一个高频对比：`std::function` 和 C 风格回调类型的差异，本质是“函数指针 + `void*` 上下文”对“对象化、带状态的统一可调用包装器”的差别，并补充捕获 lambda 不能直接当 C ABI 回调的限制。
- 新增 `04-C-Cpp/C++ std thread 入门：线程入口、join 与参数传递.md`，把 `std::thread` 放到 `std::function` 之后来讲，重点解释线程入口其实就是可调用对象、`join/detach` 的差别、参数默认按值保存、引用传参为什么常要配合 `std::ref`，以及 `lambda`/`unique_ptr` 如何把状态一起带进新线程。
- 更新 `04-C-Cpp/C++ std function 入门：lambda、回调与统一可调用类型.md` 和 `04-C-Cpp/C++ 容器与对象语义下一步学习路线.md`，把学习路线继续接到 `std::thread`，为后续 `mutex` 和并发同步建立入口。
- 更新 `04-C-Cpp/C++ std thread 入门：线程入口、join 与参数传递.md`，补充一个高频追问：`std::thread t(add_one, x)` 在 `add_one(int&)` 场景下通常不是“改副本”，而是直接编译失败；真正按引用传参通常要显式写 `std::ref(x)`。
- 更新 `04-C-Cpp/C++ std thread 入门：线程入口、join 与参数传递.md`，再补一组易混点：`std::thread t(add_one, &x)` 传的是指针语义，不是引用语义；如果形参是 `int&` 仍然对不上，只有形参改成 `int*` 时这条写法才自然成立。
- 更新 `04-C-Cpp/C++ std thread 入门：线程入口、join 与参数传递.md`，再补一个常见纠偏：函数形参是 `T&` 时，普通直接调用并不需要 `std::ref`；`std::ref` 主要是给 `std::thread`、`std::bind` 这类会先存参数再调用的包装层用来保住引用语义。
- 更新 `04-C-Cpp/C++ std thread 入门：线程入口、join 与参数传递.md`，补充一个术语纠偏：`std::thread` 默认不是“存右值”，而是把参数按值拷贝/移动进内部状态；右值/左值是表达式类别，和对象是如何被保存不是同一层概念。
- 新增 `04-C-Cpp/C++ 线程入门总览：执行流、同步与学习顺序.md`，先把线程这一章按总览梳顺：进程与线程区别、为什么要线程、`std::thread`、竞态条件、`mutex`、`condition_variable`、以及适合当前阶段的学习顺序。
- 更新 `04-C-Cpp/C++ std thread 入门：线程入口、join 与参数传递.md` 和 `04-C-Cpp/C++ 容器与对象语义下一步学习路线.md`，把线程学习入口前移到“线程总览”，形成先总览再进 `std::thread` 细节的路径。
- 新增 `04-C-Cpp/C++ 线程同步入门：竞态条件、mutex 与 lock_guard.md`，把线程学习正式接到同步层，集中讲清竞态条件为什么本质上是“时序问题”、`x++` 为什么不安全、`mutex` 实际保护的是哪段临界区，以及 `lock_guard` 作为 RAII 加锁包装为什么比手写 `lock/unlock` 更稳。
- 更新 `04-C-Cpp/C++ std thread 入门：线程入口、join 与参数传递.md`，补一个高频编译错误：线程入口 lambda 的形参与 `std::thread` 传入的实参数量/类型必须匹配；并顺手澄清“临时线程对象直接 `.detach()` 在语法上是成立的，但教学和调试阶段通常更推荐先 `join()`”。
- 新增 `04-C-Cpp/C++ 线程深入理解：执行流、共享内存与生命周期.md`，把线程往底层再拉一层：从“线程是可调度执行上下文”切入，系统梳理进程与线程的资源边界、线程自己的寄存器/栈/调度状态、`std::thread` 只是线程句柄而非线程本体、`join/detach` 的真实生命周期语义、并发与并行的区别，以及为什么 `mutex` 真正保护的是对象不变量而不是单个变量。
- 更新 `04-C-Cpp/C++ 线程深入理解：执行流、共享内存与生命周期.md`，补一个高频纠偏：同一进程内线程共享地址空间、代码段、全局区、堆和文件描述符；线程自己独享的是寄存器上下文、程序计数器、调用栈和调度状态，避免把“进程之间隔离”和“线程之间共享”这两层说反。
- 更新 `04-C-Cpp/C++ 线程深入理解：执行流、共享内存与生命周期.md`，补一个高频区分：局部变量一旦被共享给别的线程，风险通常分成两层来看，一层是生命周期失控导致的悬空引用/指针，另一层是共享可变状态在没有同步下产生的 data race；两类问题可能同时存在，不能简单混成“竞争问题”。
- 更新 `04-C-Cpp/C++ 线程同步入门：竞态条件、mutex 与 lock_guard.md`，补充 `std::mutex`、`std::lock_guard`、`std::unique_lock` 三者的层级关系，重点澄清 `unique_lock` 不是另一种“普通锁”，而是管理锁拥有权的 RAII 对象，并解释它为什么能配合 `condition_variable`。
- 新增 `04-C-Cpp/C++ RAII 入门：资源获取即初始化.md`，把 RAII 从线程锁和智能指针里抽出来单独讲清：资源责任如何绑定到对象的构造/析构、为什么“析构时自动解锁”本质上是析构函数里执行清理逻辑、以及对锁来说真正被对象管理的是“持锁时段”而不是 `mutex` 本体寿命。
- 更新 `04-C-Cpp/C++ RAII 入门：资源获取即初始化.md`，补一个关键边界：为什么局部自动对象在作用域结束时会触发析构，以及这条规则不能直接推广到 `new` 出来的堆对象，避免把“指针变量销毁”和“被指向对象析构”混为一谈。
- 更新 `04-C-Cpp/C++ RAII 入门：资源获取即初始化.md`，把 RAII 从“锁会自动解锁”继续扩成完整框架：补充为什么 C++ 需要 RAII、它如何通过对象生命周期兜底清理时机、除了锁和内存还有文件/socket/事务这些典型资源，以及它和垃圾回收在释放时机确定性上的区别。
- 更新 `04-C-Cpp/C++ RAII 入门：资源获取即初始化.md`，补一个最基础的术语纠偏：析构的主体是“对象”而不是 `class` 类型本身，并把“局部对象离开作用域时会析构”这句话单独明确下来。
- 新增 `04-C-Cpp/C++ 对象的构造顺序、析构顺序与 RAII.md`，把“构造正序、析构逆序”单独抽出来讲清：局部对象顺序、成员按声明顺序构造、初始化列表不决定成员构造顺序、基类与派生类的构造/析构先后，以及这套规则为什么能天然支撑 RAII 和异常安全。
- 更新 `04-C-Cpp/C++ 对象的构造顺序、析构顺序与 RAII.md`，补充“构造函数抛异常时到底谁会析构”这一层：外层对象自己为何不会析构、已成功构造的成员和基类为何会逆序清理、异常发生在成员构造阶段和构造函数体阶段的差别，以及 `new T()` 构造失败时对象内存与构造中手动拿到的其他资源应如何区分。
- 更新 `04-C-Cpp/C++ RAII 入门：资源获取即初始化.md` 和 `04-C-Cpp/C++ 智能指针入门：unique_ptr、shared_ptr 与 weak_ptr.md`，补一个高频纠偏：在内存管理里，RAII/智能指针不是“替你申请内存”的概念，而是“把申请出来的资源立刻纳入对象所有权与析构清理体系”，并顺手收束了普通对象、`vector/string`、`unique_ptr`、`shared_ptr` 的选择顺序。
- 更新 `04-C-Cpp/C++ RAII 入门：资源获取即初始化.md` 和 `04-C-Cpp/STL 与常用容器入门.md`，补充一个高频连接点：`vector` 本身就是 RAII 容器，不只是内部“碰巧用了动态内存”，而是它这个对象自身负责一段 buffer 及其中元素对象的构造、搬迁、销毁和最终释放。
- 更新 `04-C-Cpp/STL 与常用容器入门.md`，补充 `vector` 析构时对不同元素类型的差异：从对象语义上它都负责销毁元素，但对 `int` 这类平凡析构类型，销毁步骤通常被优化到几乎无成本；对自定义非平凡析构类型则会真实逐个调用析构函数。
- 更新 `04-C-Cpp/vector 扩容、异常安全与 noexcept.md`，补充一个高频纠偏：`int` 和 `std::string` 都是类型，真正区别在于前者常是平凡类型、后者是带对象语义的非平凡类类型；因此 `vector<int>` 扩容可以近似理解成高效搬字节，而 `vector<string>` 扩容语义上必须逐个按对象移动或拷贝并析构旧对象。
- 更新 `04-C-Cpp/拷贝构造、移动构造与 vector 扩容.md`，补充一个高频纠偏：`copy` 比 `move` 贵，关键通常不在“多析构一次”，而在是否需要真的复制资源；同时澄清“移动像改指针”只是管理堆资源类型上的近似图像，标准要求的是转移后的对象语义正确，而不是实现必须只改一个指针。
- 新增 `04-C-Cpp/C++ string 深入理解：SSO、拷贝、移动与性能直觉.md`，把 `std::string` 单独拆开讲透：对象模型、长字符串堆分配模式、短字符串 `SSO` 模式、为什么短字符串的 move 也可能只是搬对象内部的小数据，以及这件事如何改变对 `vector<string>` 扩容成本的直觉；并把新页挂回 `move` 与 `vector` 相关笔记。
