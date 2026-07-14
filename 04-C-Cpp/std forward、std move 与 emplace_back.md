# `std::forward`、`std::move` 与 `emplace_back`

## 一句话先理解 `emplace_back`

`emplace_back` 是：

- 在容器尾部**原地构造元素**的接口
- 参数会被直接转发给元素类型 `T` 的构造函数

最典型的形式是：

```cpp
std::vector<Person> v;
v.emplace_back("Tom", 18);
```

这里不是先在外面造一个 `Person`，再塞进 `vector`；
而是直接在 `vector` 尾部那块位置上构造出这个 `Person`。

## 为什么这页重要

这 3 个点是对象语义和容器接口真正落地的地方：

- `std::move`：强制把对象当成可移动值
- `std::forward`：保留调用方原本的值类别
- `emplace_back`：靠完美转发把参数原地交给元素构造函数

## 1. `std::move(x)` 和 `std::forward<T>(x)` 的本质区别

### `std::move(x)`

它的语义是：

- 不管 `x` 原来是什么
- 我现在都要把它当成可移动对象

所以它是：

- **无条件转换**

你可以把它理解成：

- “我不保留原语义了，我就是要把它当右值处理”

### `std::forward<T>(x)`

它的语义是：

- 如果原始实参是左值，就继续按左值传
- 如果原始实参是右值，就继续按右值传

所以它是：

- **按模板推导结果有条件地保留原始值类别**

你可以把它理解成：

- “我不是要强行 move，我只是原样转发”

## 2. 为什么模板里不用 `std::move(x)`，而要用 `std::forward<T>(x)`

看这个包装函数：

```cpp
template <class T>
void wrapper(T&& x) {
    target(std::move(x));
}
```

问题是：

- 不管调用方传左值还是右值
- 你都把它强行当右值送给 `target`

例如：

```cpp
std::string s = "hi";
wrapper(s);
```

调用方传的是左值，但你在 wrapper 里一旦写 `std::move(x)`，就把它偷成右值了。

这通常不是我们想要的。

正确写法是：

```cpp
template <class T>
void wrapper(T&& x) {
    target(std::forward<T>(x));
}
```

这样：

- 传左值进来，`target` 收到左值
- 传右值进来，`target` 收到右值

这才叫完美转发。

## 3. 为什么“有名字的右值引用变量本身是左值”

这是最容易卡人的点。

看：

```cpp
std::string&& r = std::string("hi");
```

这里：

- `r` 的类型是右值引用

但：

- 表达式 `r` 本身是左值

原因很简单：

- 只要一个对象有名字，你就可以反复引用它
- 它就有稳定身份
- 所以在表达式层面它按左值处理

这就是为什么：

```cpp
target(r);
```

不会自动把 `r` 当右值传下去。

如果你真想继续把它当可移动值传递，你得显式写：

```cpp
target(std::move(r));
```

或者在转发场景里写：

```cpp
target(std::forward<T>(x));
```

## 4. `push_back` 和 `emplace_back` 在这里怎么落地

### `push_back`

大致有两种常见入口：

```cpp
v.push_back(x);            // 左值
v.push_back(std::move(x)); // 右值
```

它的特点是：

- 你先已经有一个对象 `x`
- 再把这个对象交给 `vector`

所以它更像：

- “把现成对象放进去”

### `emplace_back`

```cpp
v.emplace_back(args...);
```

它的特点是：

- 你不一定先构造一个完整对象
- 而是把构造参数直接交给容器
- 容器在尾部原地构造元素

所以它更像：

- “把构造这个对象所需的参数交进去”

## 5. `emplace_back` 为什么离不开完美转发

一个典型实现思路会长这样：

```cpp
template <class... Args>
reference emplace_back(Args&&... args) {
    // 在尾部原地构造元素
    new (end_) T(std::forward<Args>(args)...);
}
```

这里的关键是：

- `Args&&...` 是一组 forwarding references
- `std::forward<Args>(args)...` 保留每个参数原本的左值/右值属性

如果不用 `forward`，而直接写：

```cpp
T(args...)
```

那所有有名字的 `args` 在函数体里都会按左值处理。  
这样原本传进来的右值语义就丢了。

如果全写成：

```cpp
T(std::move(args)...)
```

那又会把所有参数无脑当右值，连原本的左值也被强行 move 了。

所以 `emplace_back` 最需要的就是：

- 既通用
- 又不破坏原始值类别

这正是完美转发存在的原因。

## 6. 一个很短的对比

### `push_back`

- 对象先存在
- 再放进容器

### `emplace_back`

- 参数先进容器接口
- 容器内部直接构造对象

### `std::move`

- 无条件把对象当右值

### `std::forward`

- 按原始值类别继续传递

## 7. 一句话收住

`std::move` 是“我要把你当成可移动值”，`std::forward` 是“我要把你按原样继续传下去”，而 `emplace_back` 则是标准库中完美转发最典型的落地点之一。

## 8. `emplace_back` 的真实代码路径怎么串起来

可以先把它想成一个非常简化的版本：

```cpp
template<class... Args>
T& my_emplace_back(Args&&... args) {
    if (size_ == capacity_) {
        grow();  // 必要时先扩容
    }

    new (end_) T(std::forward<Args>(args)...);
    ++end_;
    return *(end_ - 1);
}
```

你现在重点看最后这一行：

```cpp
T(std::forward<Args>(args)...)
```

它做的事情是：

1. `Args&&...` 先把所有参数接进来
2. 每个参数在函数体里其实都有名字，所以默认都成了左值表达式
3. `std::forward<Args>(args)...` 再把它们恢复成“原来传进来时的值类别”
4. 最后把这些参数原样交给 `T` 的构造函数

所以 `emplace_back` 的本质不是“更高级的 push_back”，而是：

- **把构造参数直接转发给元素类型 `T` 的构造函数**

## 9. 一个具体例子

先看元素类型：

```cpp
struct Person {
    std::string name;
    int age;

    Person(std::string n, int a)
        : name(std::move(n)), age(a) {}
};
```

再看调用：

```cpp
std::vector<Person> v;
std::string s = "Tom";

v.emplace_back(s, 18);
```

这里会发生：

1. `Args...` 推导成：
   - `std::string&`
   - `int`
2. 参数 `args...` 在函数体里都有名字，本来都会按左值看
3. `std::forward<Args>(args)...` 把第一个参数继续按左值传，把第二个按右值/值传
4. 最终调用的是：

```cpp
Person(std::string, int)
```

如果你写：

```cpp
v.emplace_back(std::move(s), 18);
```

那第一项就会按右值转发，给 `Person` 的构造函数更多移动机会。

## 10. 如果不用 `forward` 会怎样

### 直接写 `args...`

```cpp
new (end_) T(args...);
```

问题：

- 所有 `args` 都是有名字的
- 所以它们全都会按左值处理

结果：

- 原本调用方传进来的右值语义丢了

### 全写 `std::move(args)...`

```cpp
new (end_) T(std::move(args)...);
```

问题：

- 所有参数都会被强行当右值
- 连原本传进来的左值也被偷成右值

结果：

- 包装层破坏了调用方语义

所以 `forward` 正好卡在中间：

- 不丢右值
- 不误伤左值

## 11. 和 `push_back` 的真实差别

### `push_back`

```cpp
Person p("Tom", 18);
v.push_back(p);
```

这里流程更像：

1. 先有一个现成的 `Person` 对象 `p`
2. 再把 `p` 拷贝/移动进容器

### `emplace_back`

```cpp
v.emplace_back("Tom", 18);
```

这里流程更像：

1. 直接把 `"Tom"` 和 `18` 交给容器
2. 容器内部在尾部原地构造 `Person`

所以 `emplace_back` 的优势不在于“永远更快”，而在于：

- 某些场景可以少一层中间对象
- 参数可以直接落到目标构造函数上

## 12. 这一节最该记住的因果链

1. `Args&&...` 在模板里是 forwarding references
2. `args...` 一旦有名字，在函数体里全是左值表达式
3. `std::forward<Args>(args)...` 用来恢复原始值类别
4. `emplace_back` 靠这套机制把参数精准送进元素构造函数

## 13. 从底层和 OS 角度看，`emplace_back` 到底快在哪里

先说结论：

- **通常不是 OS 层面快**
- **主要是 C++ 对象构造路径更短**

### OS 层面通常没区别

无论你写：

```cpp
v.push_back(T(args...));
```

还是：

```cpp
v.emplace_back(args...);
```

如果这次没有触发扩容，那：

- 都不会找操作系统再申请内存页
- 都只是用 `vector` 已有缓冲区尾部那一小块空间

所以多数时候根本谈不上“OS 更快”。

只有当：

- `size == capacity`
- 触发扩容

这时才会进入更重的内存分配路径。  
但这一点对 `push_back` 和 `emplace_back` 都一样，不是两者区别的核心。

### 真正可能更快的地方：少一层中间对象

看这两种写法：

```cpp
v.push_back(T(args...));
v.emplace_back(args...);
```

第一种更像：

1. 先在别处构造一个临时 `T`
2. 再把这个临时对象拷贝/移动到 `vector` 尾部

第二种更像：

1. 直接在 `vector` 尾部那块内存上构造 `T`

所以 `emplace_back` 潜在更快，主要是因为：

- 少一次中间对象路径
- 少一次额外的拷贝/移动机会

### 从机器执行角度看，省的是这些

可能省掉：

- 一次临时对象构造
- 一次移动构造或拷贝构造
- 一次临时对象析构

不一定每次都全省，但“优化空间”主要在这里。

### 什么时候差别最明显

当元素类型：

- 构造代价明显
- 移动也不是零成本
- 你传进去的是构造参数而不是现成对象

这时 `emplace_back` 更可能占优。

### 什么时候差别不大

1. 元素类型很轻，比如 `int`
2. 编译器已经把中间临时优化得很好
3. 你本来就已经有一个现成对象

比如：

```cpp
T obj(...);
v.push_back(std::move(obj));
```

这时 `emplace_back` 往往没有你想象中那么神。

## 14. 一句话总结

`emplace_back` 的“快”主要来自对象构造路径更直接，而不是 OS 层面有什么特殊优化；它本质上优化的是“中间对象和额外构造/移动”，不是系统调用。

## 15. 面试题：`std::forward` 和 `std::move` 的功能分别是什么

一个够用的面试回答可以直接说：

- `std::move(x)`：**无条件**把 `x` 转成右值表达式，意思是“后面可以把它当成可移动对象”
- `std::forward<T>(x)`：**按原始实参的值类别转发**，左值继续传左值，右值继续传右值

如果再往下补一层本质：

- 它们本质上都主要是**类型转换工具**
- 自己并不搬资源
- 真正搬资源的是后面被调用的移动构造、移动赋值或右值重载

所以一句话区分就是：

- `move`：我现在就要把它当右值
- `forward`：我只是尽量按原样继续往后传

## 16. 面试里最容易追问的点：它们什么时候会“无用”

这里的“无用”通常不是说代码报错，  
而是说：

- 写了也没有性能收益
- 或者根本没达到你以为的语义效果

### `std::move` 常见“无用”场景

#### 场景 1：对象是 `const`

```cpp
const std::string s = "abc";
std::string x = std::move(s);
```

这里 `std::move(s)` 的结果是：

- `const std::string&&`

但大多数移动构造要的是：

- `std::string&&`

因为源对象是 `const`，标准库通常不能改它，所以最后往往还是走拷贝。  
也就是说：

- **语法上写了 `move`**
- **语义上没真正移动**

这是面试里非常经典的“`move` 无用”例子。

#### 场景 2：类型根本没有高效移动

比如：

- 类型没写移动构造
- 或者移动构造被删除了
- 或者它的“移动构造”内部其实还是深拷贝

这时即使你写：

```cpp
std::move(x)
```

最后也可能：

- 退回拷贝
- 或者虽然走了 move 语法，但性能并没更好

#### 场景 3：基础类型或特别轻的类型

比如：

```cpp
int a = 1;
int b = std::move(a);
```

对 `int` 这种类型，拷贝和移动几乎没区别。  
所以这里 `move` 基本没有实际价值。

#### 场景 4：`return std::move(local)`

```cpp
std::string f() {
    std::string s = "abc";
    return std::move(s);
}
```

很多时候这里是没必要的，因为：

- 编译器本来就可能做返回值优化
- 你手动 `move` 反而可能让代码更丑

面试里答到这层就够了：

- **返回局部对象时，不要下意识乱写 `std::move`**
- **先让编译器做返回值优化**

### `std::forward` 常见“无用”场景

#### 场景 1：不在转发场景里

`std::forward` 最有意义的地方是：

```cpp
template<class T>
void wrapper(T&& x) {
    target(std::forward<T>(x));
}
```

也就是：

- `T` 来自模板推导
- 参数是 forwarding reference

如果不是这种场景，`forward` 往往没有存在意义。

比如：

```cpp
void f(std::string&& s) {
    g(std::forward<std::string>(s));
}
```

这里 `T` 不是“按实参推导出来再保留值类别”的那个角色，  
你写 `forward<std::string>(s)`，效果基本就等于：

```cpp
g(std::move(s));
```

所以这里用 `forward` 没有它真正该有的语义价值。

#### 场景 2：参数本来就是普通左值引用

```cpp
template<class T>
void f(T& x) {
    g(std::forward<T>(x));
}
```

这里 `x` 不是什么 forwarding reference，  
它本来就只能按左值这条线走。

所以这个 `forward` 通常也没有“保留左右值两种可能性”的意义。

#### 场景 3：你明明就是想偷资源

如果你的意图就是：

- 我后面不再用这个对象
- 我现在就要把它当右值交出去

那就该直接 `std::move`，而不是 `std::forward`。  
因为 `forward` 的重点不是“更高级”，而是“保留原语义”。

## 17. 一个最适合面试收尾的总结

你可以这样收住：

`std::move` 是强制把对象转成右值表达式，给后续移动语义创造机会；`std::forward` 是在模板转发场景里保留原始值类别。它们都会在一些场景下“看起来写了但没收益”，最典型的是 `move const 对象` 和 `forward` 用在非 forwarding reference 场景里。

## 18. 你最好顺手能举出的两个例子

### 例子 1：`move` 对 `const` 无效

```cpp
const std::string s = "abc";
std::string x = std::move(s); // 往往还是拷贝
```

### 例子 2：`forward` 只在完美转发里最有意义

```cpp
template<class T>
void wrapper(T&& x) {
    target(std::forward<T>(x));
}
```

因为这里它才能真正做到：

- 传左值就继续传左值
- 传右值就继续传右值

## 19. 和“无用”不同：`std::move` / `std::forward` 的常见误用

“无用”是：

- 你写了，但没收益

“误用”是：

- 你把语义写错了
- 可能导致对象被提前偷走
- 或者包装层破坏调用方原意

### 误用 1：对后面还要继续用的对象乱 `move`

```cpp
std::string s = "abc";
vec.push_back(std::move(s));
std::cout << s << '\n';
```

这不一定立刻报错，但语义上已经危险了。  
因为 `s` 被当成可移动对象交出去了，后面虽然还能用，但只保证“有效未指定状态”。

面试里要会说：

- **`move` 不是复制后保留原值**
- **而是表明“这个对象的资源可以被拿走”**

### 误用 2：在 forwarding reference 包装层里把 `forward` 写成 `move`

```cpp
template<class T>
void wrapper(T&& x) {
    target(std::move(x));
}
```

这会把：

- 调用方传来的左值

也强行偷成右值。

正确写法应该是：

```cpp
template<class T>
void wrapper(T&& x) {
    target(std::forward<T>(x));
}
```

这是 `move` / `forward` 最经典的误用区别。

### 误用 3：把 `forward` 当成“更高级的 move”

有些人会觉得：

- `forward` 比 `move` 更现代
- 所以能写 `forward` 就都写 `forward`

这是错的。

`forward` 不是拿来替代 `move` 的，它只适合：

- 模板推导
- forwarding reference
- 保留原始值类别

如果你本来就明确要转移资源，那就该写 `move`。

### 误用 4：对同一个对象 `move` 之后，还按原值语义继续依赖它

```cpp
std::string s = "hello";
auto t = std::move(s);

if (s == "hello") {   // 这种期待就是错的
    ...
}
```

这里错不在“不能访问 `s`”，  
而在于：

- 你不能再假设它还保留原内容

### 误用 5：函数参数一进来就无脑 `std::move`

```cpp
void set_name(std::string s) {
    name_ = std::move(s);
}
```

这个例子本身不一定错，  
因为这里参数 `s` 是按值传进来的局部副本，函数里再 `move` 往往是合理的。

真正的误区是很多人把这种写法机械套到别的地方，以为：

- “只要写 `move` 就更高效”

其实要先看：

- 这个对象是不是你自己的局部副本
- 后面还要不要继续用它
- 这里是不是转发场景

## 20. 面试里可以怎么收住“误用”

你可以直接说：

`std::move` 最常见的误用是对后面还要依赖原值的对象乱 move，或者在模板包装层里把本该 `forward` 的地方写成了 `move`；`std::forward` 最常见的误用是脱离 forwarding reference 场景乱用，把它当成更高级的 `move`。

## 21. 一个很容易记错的点：`forward` 不是“模板里声明右值时才用”

更准确地说：

- `std::forward` 主要用在**模板转发场景**
- 尤其是参数写成 `T&&`，并且这里的 `T` 是**模板推导出来的**
- 也就是参数实际上是 **forwarding reference**

这时如果你要把参数继续传给下一个函数，就用：

```cpp
template<class T>
void wrapper(T&& x) {
    target(std::forward<T>(x));
}
```

因为你想保留的是：

- 原来传进来是左值，就继续按左值传
- 原来传进来是右值，就继续按右值传

所以 `forward` 不是拿来“声明右值”的，  
而是拿来：

- **在模板里保留原始值类别**

## 22. 和普通右值引用对比一下

### 情况 1：普通右值引用

```cpp
void f(std::string&& s) {
    g(std::move(s));
}
```

这里通常更像是：

- 你已经明确知道 `s` 这条线该按右值处理
- 所以更常见的是 `move`

### 情况 2：模板 forwarding reference

```cpp
template<class T>
void f(T&& x) {
    g(std::forward<T>(x));
}
```

这里才是 `forward` 最典型的舞台。  
因为你此时不知道调用方传进来的是左值还是右值，你要做的是：

- 不偷左值
- 不丢右值

## 23. 一句话纠正你的这句理解

不要记成：

- “`forward` 是模板声明右值的时候再用”

更该记成：

- “`forward` 是模板里保留实参原始左/右值属性的时候用”

## 相关页面

- [[04-C-Cpp/T&&、引用折叠与完美转发]]
- [[04-C-Cpp/左值、右值与 std move]]
- [[04-C-Cpp/STL 与常用容器入门]]
