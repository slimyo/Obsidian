## C++ Lambda 表达式详解

Lambda 表达式是 C++11 引入的一种"匿名函数"，它让你可以在代码中就地定义一个函数对象，常用于 STL 算法、回调、异步任务等场景。

### 一、基本语法

完整形式是：

cpp

```cpp
[capture](parameters) specifiers -> return_type { body }
```

各部分含义：

- `[capture]` 捕获列表（必需）：从外部作用域捕获哪些变量
- `(parameters)` 参数列表（可省略）：和普通函数一样
- `specifiers` 说明符（可省略）：`mutable`、`constexpr`、`noexcept` 等
- `-> return_type` 返回类型（可省略）：编译器一般能自动推导
- `{ body }` 函数体（必需）

最简单的例子：

cpp

```cpp
auto greet = []() { std::cout << "Hello\n"; };
greet();  // 调用

auto add = [](int a, int b) { return a + b; };
std::cout << add(3, 5);  // 8
```

### 二、捕获列表（最关键的部分）

捕获列表决定 lambda 能访问哪些外部变量、以什么方式访问。

**按值捕获 `[x]`**：拷贝一份，lambda 内部用的是副本。

cpp

```cpp
int x = 10;
auto f = [x]() { std::cout << x; };
x = 20;
f();  // 输出 10，因为捕获时已拷贝
```

**按引用捕获 `[&x]`**：使用原变量的引用，外部变化会影响 lambda 内的值。

cpp

```cpp
int x = 10;
auto f = [&x]() { std::cout << x; };
x = 20;
f();  // 输出 20
```

**默认捕获 `[=]` 和 `[&]`**：分别表示"用到的所有外部变量都按值/按引用捕获"。

cpp

```cpp
int a = 1, b = 2;
auto f1 = [=]() { return a + b; };  // a 和 b 都按值
auto f2 = [&]() { a++; b++; };       // a 和 b 都按引用
```

**混合捕获**：默认 + 例外。

cpp

```cpp
int a = 1, b = 2, c = 3;
auto f = [=, &c]() { /* a, b 按值；c 按引用 */ };
auto g = [&, a]()  { /* 默认按引用，但 a 按值 */ };
```

**捕获 `this`**：在成员函数内部可以捕获 `this` 指针来访问类成员。

cpp

```cpp
class Counter {
    int count = 0;
public:
    auto make_incrementer() {
        return [this]() { ++count; };  // 捕获 this 指针
        // C++17 起可以用 [*this] 按值捕获整个对象
    }
};
```

注意：`[=]` 在 C++20 之前会隐式捕获 `this`，但 C++20 已弃用此行为，应显式写 `[=, this]`。

### 三、初始化捕获（C++14）

也叫"广义捕获"，可以在捕获列表里创建新变量、移动捕获。

cpp

```cpp
// 移动捕获（unique_ptr 不能拷贝，必须移动）
auto ptr = std::make_unique<int>(42);
auto f = [p = std::move(ptr)]() { std::cout << *p; };

// 计算后捕获
int x = 10;
auto g = [y = x * 2]() { return y; };  // 捕获 y = 20
```

### 四、mutable 说明符

按值捕获的变量在 lambda 内部默认是 `const` 的，不能修改。加上 `mutable` 才允许修改副本（不影响外部）。

cpp

```cpp
int x = 10;
auto f = [x]() mutable { x++; std::cout << x; };
f();  // 11
f();  // 12（副本被保留在 lambda 对象内）
std::cout << x;  // 外部 x 仍是 10
```

### 五、返回类型

通常自动推导，但函数体复杂或有多种返回路径时可能需要显式指定：

cpp

```cpp
auto f = [](int x) -> double {
    if (x > 0) return x;     // int
    else       return x / 2.0;  // double
    // 不写 -> double 时编译器会报错（推导冲突）
};
```

### 六、泛型 Lambda（C++14）

参数类型可以用 `auto`，相当于一个模板：

cpp

```cpp
auto add = [](auto a, auto b) { return a + b; };
add(1, 2);        // int
add(1.5, 2.5);    // double
add(std::string("a"), std::string("b"));  // string
```

C++20 引入了显式模板参数语法：

cpp

```cpp
auto f = []<typename T>(std::vector<T>& v) {
    return v.size();
};
```

### 七、Lambda 的本质

Lambda 实际上是编译器生成的一个匿名类（带有 `operator()` 的"闭包类型"）。所以：

cpp

```cpp
auto f = [](int x) { return x * 2; };
// 大致等价于：
struct __Anonymous {
    int operator()(int x) const { return x * 2; }
};
auto f = __Anonymous{};
```

每个 lambda 的类型都是唯一且不可命名的，所以通常只能用 `auto` 接收。如果需要存储不同 lambda 或函数指针，可用 `std::function`：

cpp

```cpp
std::function<int(int)> fn = [](int x) { return x * 2; };
```

但 `std::function` 有类型擦除的开销，在性能敏感场景优先用 `auto` 或模板参数。

无捕获的 lambda 还可以隐式转换为函数指针：

cpp

```cpp
int (*fp)(int) = [](int x) { return x * 2; };  // OK
```

### 八、常见用法

**配合 STL 算法**：

cpp

```cpp
std::vector<int> v = {3, 1, 4, 1, 5, 9, 2, 6};

std::sort(v.begin(), v.end(), [](int a, int b) { return a > b; });

auto it = std::find_if(v.begin(), v.end(), [](int x) { return x > 4; });

int count = std::count_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; });
```

**立即执行（IIFE）用于复杂初始化**：

cpp

```cpp
const auto config = []() {
    Config c;
    c.load("file.json");
    c.validate();
    return c;
}();  // 注意末尾的 () 直接调用
```

**递归 lambda**（C++14 起常用 `auto` + 自引用）：

cpp

```cpp
auto factorial = [](auto self, int n) -> int {
    return n <= 1 ? 1 : n * self(self, n - 1);
};
factorial(factorial, 5);  // 120


```

- C++23 起更优雅[[lambda#cpp23]]：使用 `deducing this`
```cpp
auto factorial = [](this auto self, int n) -> int { ... };
```

### 九、常见陷阱

**悬空引用**：lambda 跨越作用域使用，引用捕获的变量已销毁。

cpp

```cpp
auto make_lambda() {
    int local = 42;
    return [&local]() { return local; };  // ⚠️ 危险：local 会销毁
}
auto f = make_lambda();
f();  // 未定义行为
```

应改为按值捕获 `[local]` 或初始化捕获。

**默认按引用捕获的滥用**：在成员函数中 `[&]` 容易隐式持有 `this`，回调延迟执行时可能 `this` 已失效。

**捕获大对象**：按值捕获大对象会触发拷贝，按需用 `[obj = std::move(obj)]` 移动捕获。

---

掌握 lambda 的关键就两点：**理解捕获列表的语义**（值/引用/移动），以及**理解它本质是个匿名类对象**。剩下的语法在用的时候查一查就行。如果你想看某个具体场景（比如异步、协程、模板元编程里的用法），告诉我我再展开。

---

### lambda何时可以用函数指针接收

**只有"无捕获"的 lambda** 才能转换为函数指针。你这个例子捕获列表是空的 `[]`，所以完全可以：

cpp

```cpp
void (*greet)() = []() { std::cout << "Hello\n"; };
greet();  // OK
```

C++ 标准明确规定：无状态（无捕获）的 lambda 存在一个隐式转换函数，能转成对应签名的普通函数指针。

### 有捕获就不行了

一旦捕获列表非空，lambda 就携带了状态（相当于一个有成员变量的对象），普通函数指针无法表示这种状态：

cpp

```cpp
int x = 10;
void (*f)() = [x]() { std::cout << x; };  // ❌ 编译错误
```

因为 `void(*)()` 只是一个地址，没地方存 `x`。

### 那有捕获时怎么显式声明？

每个 lambda 的真实类型是编译器生成的**匿名闭包类型**，你根本写不出来。所以有两种选择：

**1. 用 `std::function`（类型擦除，能容纳任何可调用对象）：**

cpp

```cpp
#include <functional>

int x = 10;
std::function<void()> greet = [x]() { std::cout << x; };
```

代价：有堆分配和虚调用的开销，性能敏感时要注意。

**2. 用 `decltype` 推导：**

cpp

```cpp
auto lambda = [x]() { std::cout << x; };
decltype(lambda) another = lambda;  // 拷贝同类型
```

但这其实和 `auto` 没本质区别，只是换种写法。

### 三种方式的对比

cpp

```cpp
// 1. 函数指针：仅限无捕获，零开销，可与 C 代码互操作
void (*fp)()              = []() { /*...*/ };

// 2. std::function：通用，有开销
std::function<void()> sf  = [x]() { /*...*/ };

// 3. auto：编译期已知确切类型，零开销，最常用
auto lam                  = [x]() { /*...*/ };
```

### 一个实用场景

C 风格 API（比如 `qsort`、pthread、各种回调）只接受函数指针，这时无捕获 lambda 就特别有用：

cpp

```cpp
std::qsort(arr, n, sizeof(int),
    [](const void* a, const void* b) -> int {
        return *(int*)a - *(int*)b;
    });
```

如果你给这个 lambda 加了捕获，编译就会失败——这也是为什么很多 C API 设计成接受一个 `void* userdata` 参数，让用户手动传递"捕获的状态"。

---

# cpp23


这个特性叫 **"显式对象参数"**（explicit object parameter），也常被称为 "deducing this"，是 C++23 的重要新特性。

### 先回顾问题：为什么递归 lambda 这么别扭

C++14/17 写递归 lambda 必须把自己作为参数传进去：

cpp

```cpp
auto factorial = [](auto self, int n) -> int {
    return n <= 1 ? 1 : n * self(self, n - 1);
    //                      ^^^^ 还得手动把 self 再传一次
};
factorial(factorial, 5);  // 调用时也得传两次
```

为什么要这么写？因为 lambda 在定义时**还没有名字**——`auto factorial = ...` 这个赋值要等右边的 lambda 完全构造出来才发生，所以 lambda 体内不能引用 `factorial`（即使引用了，也得按引用捕获 `factorial` 这个变量本身，又会牵扯到生命周期问题）。

变通办法就是把"自己"作为第一个参数，用 `auto` 让编译器推导出 lambda 的类型。难看，调用也累赘。

### C++23 的解法：`this auto self`

cpp

```cpp
auto factorial = [](this auto self, int n) -> int {
    return n <= 1 ? 1 : n * self(n - 1);
    //                      ^^^^ 直接调用，不用再传 self
};
factorial(5);  // 调用也只传一个参数
```

关键点：

**`this auto self` 是一个特殊的"显式对象参数"**。它必须是参数列表里的第一个参数，前面带 `this` 关键字。它代表"调用这个 lambda/成员函数所用的那个对象本身"——也就是 lambda 自己。

**`self` 就是 lambda 对象的引用**（具体类型由编译器推导），所以 `self(n-1)` 直接就是在调用 lambda 自己。

**调用方不需要传它**。`this` 参数是隐式传递的，就像普通成员函数里的 `this` 指针你不用手动传一样。

### 对比一下

cpp

```cpp
// C++14 方式：丑陋
auto fact14 = [](auto self, int n) -> int {
    return n <= 1 ? 1 : n * self(self, n - 1);
};
fact14(fact14, 5);

// C++23 方式：优雅
auto fact23 = [](this auto self, int n) -> int {
    return n <= 1 ? 1 : n * self(n - 1);
};
fact23(5);
```

调用语法干净了，递归调用也不用再传 `self` 给自己。

### 这个特性不止用于 lambda

`deducing this` 其实是个更通用的特性，普通成员函数也能用，主要用途包括：

**1. 消除 const/非 const 重载的重复代码**

以前要写两份：

cpp

```cpp
class Container {
    T& at(size_t i)             { /* 一堆逻辑 */ }
    const T& at(size_t i) const { /* 同样的逻辑，复制一遍 */ }
};
```

C++23 一份搞定：

cpp

```cpp
class Container {
    auto& at(this auto& self, size_t i) {
        // self 自动推导为 Container& 或 const Container&
        return self.data[i];
    }
};
```

**2. 根据值类别（左值/右值）做不同处理**

cpp

```cpp
class String {
public:
    auto substr(this auto&& self, size_t pos) {
        if constexpr (std::is_rvalue_reference_v<decltype(self)>) {
            // 是右值，可以move内部数据，避免拷贝
        } else {
            // 是左值，老老实实拷贝
        }
    }
};
```

**3. CRTP 的简化**——以前用 CRTP 模式要写 `static_cast<Derived*>(this)`，现在 `self` 自动就是派生类类型。

---

# 递归时的写法：auto/auto &/auto &&

### 三种写法

显式对象参数和普通参数一样，可以用三种形式：

cpp

```cpp
[](this auto self, ...)    // 按值：每次递归都拷贝一份 lambda 对象
[](this auto& self, ...)   // 按左值引用
[](this auto&& self, ...)  // 转发引用（推荐）
```

它们在功能上都能递归，但在**性能**和**适用范围**上有区别。

### 为什么 `auto&&` 是首选

**1. 避免拷贝 lambda 对象**

lambda 本质是个对象，捕获的东西就是它的成员。每次递归调用时：

- `auto self`：把整个 lambda 对象**拷贝**一份给 `self`
- `auto&& self`：直接绑定到原对象，零开销

你这个例子是 `[&]` 按引用捕获，捕获的成员只是几个引用，拷贝代价小但仍然多余。如果 lambda 按值捕获了大对象（比如一个 `std::vector` 或 `std::string`），那拷贝就很贵了：

cpp

```cpp
std::vector<int> big_data(10000);
auto f = [big_data](this auto self, int n) -> int {  // ⚠️ 每次递归拷贝整个 vector
    return n == 0 ? 0 : self(n - 1);
};

auto g = [big_data](this auto&& self, int n) -> int {  // ✅ 零拷贝
    return n == 0 ? 0 : self(n - 1);
};
```

**2. 兼容不可拷贝的 lambda**

如果 lambda 捕获了 `unique_ptr` 等只能移动的类型，lambda 本身就不可拷贝。这时按值的 `auto self` 直接编译失败：

cpp

```cpp
auto ptr = std::make_unique<int>(42);
auto f = [p = std::move(ptr)](this auto self, int n) -> void {
    //                         ^^^^^^^^^ ❌ 编译错误：lambda 不可拷贝
    if (n > 0) self(n - 1);
};

auto g = [p = std::move(ptr)](this auto&& self, int n) -> void {
    //                         ^^^^^^^^^^^ ✅ OK
    if (n > 0) self(n - 1);
};
```

**3. `mutable` 状态的语义差异**

如果 lambda 是 `mutable` 的且在递归中要修改自身状态，按值会让每层递归看到独立的副本：

cpp

```cpp
auto f = [count = 0](this auto self, int n) mutable -> void {
    ++count;  // 每层递归都是新副本，count 都从 0 开始
    if (n > 0) self(n - 1);
};

auto g = [count = 0](this auto&& self, int n) mutable -> void {
    ++count;  // 所有递归共享同一个 count
    if (n > 0) self(n - 1);
};
```

### 回到你的例子

cpp

```cpp
fs = [&](this auto&& dfs, TreeNode* node, string path) -> void { ... };
```

DFS 树遍历常常递归很深（树有几百层就调用几百次），每次拷贝一遍 lambda 对象完全没必要。用 `auto&&` 就是**惯用写法**，相当于：

> "递归调用我自己时，直接用我本人，不要拷贝。"

### 为什么是 `&&` 不是 `&`

`auto&&` 是**转发引用**（forwarding reference），既能绑定左值也能绑定右值。虽然实际中 lambda 一般是存在变量里被当作左值调用，`auto&` 也够用，但 `auto&&` 更通用，是社区约定俗成的写法——和 `for (auto&& x : range)` 同理，"不知道用什么就用 `auto&&` 准没错"。

### 一句话总结

`auto self` 是按值，会拷贝；`auto&& self` 是引用，零开销且能处理不可拷贝的 lambda。**递归 lambda 用 `auto&&` 是最佳实践**，我之前写的 `auto self` 在功能上没错，但在性能和通用性上不如 `auto&&`。