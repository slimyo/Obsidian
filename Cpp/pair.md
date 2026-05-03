# C++ pair 核心 API 速查表

> 头文件：`#include <utility>`
> 
> 本质：**固定大小为 2 的聚合类型**

---

## 一、增 / 创建（构造 / 初始化）

|操作|语法|示例|说明|
|---|---|---|---|
|**直接构造**​|`pair<T1,T2>(a,b)`|`p = std::pair<int,string>(1,"ok");`|最基础构造|
|**make_pair**​|`make_pair(a,b)`|`p = std::make_pair(1,"ok");`|自动推导类型，最常用|
|**花括号初始化**​|`{a,b}`|`p = {1,"ok"};`|C++11，简洁直观|
|**拷贝构造**​|`pair(const pair&)`|`auto p2 = p1;`|深拷贝|
|**移动构造**​|`pair(pair&&)`|`auto p2 = std::move(p1);`|C++11，转移资源|
|**默认构造**​|`pair()`|`std::pair<int,int> p;`|值初始化（基本类型为随机值）|

---

## 二、删（生命周期管理）

|操作|语法|示例|说明|
|---|---|---|---|
|**析构**​|`~pair()`|（自动调用）|销毁对象|
|**赋值覆盖**​|`operator=`|`p1 = p2;`|拷贝赋值|
|**移动赋值**​|`operator=(pair&&)`|`p1 = std::move(p2);`|C++11|

> ⚠️ `pair`本身**没有 `clear()`/ `erase()`之类接口**，
> 
> 因为其长度是固定的（始终 2 个元素）。

---

## 三、查 / 取（访问元素）

|操作|语法|示例|说明|
|---|---|---|---|
|**成员访问**​|`first / second`|`x = p.first; y = p.second;`|最常用|
|**引用访问**​|`first / second`|`p.first = 10;`|可直接修改|
|**结构化绑定**​|`auto [a,b]`|`auto [id, name] = p;`|C++17，强烈推荐|
|**指针访问**​|`&p.first`|`int* x = &p.first;`|可用于函数参数|
|**比较大小**​|`== != < > <= >=`|`if (p1 == p2)`|字典序比较（先 first，再 second）|

---

## 四、改（修改 / 交换）

|操作|语法|示例|说明|
|---|---|---|---|
|**修改成员**​|`p.first / p.second`|`p.second = "hello";`|直接赋值|
|**整体赋值**​|`operator=`|`p = {2,"xx"};`|覆盖整个 pair|
|**交换内容**​|`swap(pair&)`|`p1.swap(p2);`|O(1)，高效|
|**std::swap**​|`std::swap(p1,p2)`|`std::swap(p1,p2);`|等价于成员 swap|

---

## 五、容易忘记的重要 API / 用法

|操作|语法|示例|为什么容易忘|
|---|---|---|---|
|**忽略返回值**​|`std::ignore`|`std::tie(x, std::ignore) = p;`|只取其中一个成员|
|**函数返回 pair**​|`return {a,b};`|`return std::make_pair(a,b);`|常用于 map / 算法|
|**map 中的 value_type**​|`pair<const K,V>`|`m.begin()->first`|map 的 value 本质是 pair|
|**tuple 对比**​|`pair vs tuple`|`tuple<int,int,int>`|pair 只有两个元素|
|**const pair 访问**​|`const T& first`|`const auto& p = m.begin()->first;`|防止误修改 key|

---

## 六、实用技巧（记住这些）

### 1. 函数返回多个值（最经典用法）

```cpp
std::pair<int, int> div(int a, int b) {
    return {a / b, a % b};
}

auto [q, r] = div(10, 3);
```

---

### 2. 与 `map`/ `unordered_map`配合

```cpp
std::map<int, std::string> m;

// insert 本质是 pair
m.insert(std::make_pair(1, "one"));

// 遍历 map
for (const auto& [k, v] : m) {
    std::cout << k << ":" << v << "\n";
}
```

---

### 3. 使用 `tie`解包（C++11–C++14）

```cpp
int x, y;
std::tie(x, y) = std::make_pair(3, 4);
```

---

### 4. 自定义比较（如 priority_queue）

```cpp
struct cmp {
    bool operator()(const std::pair<int,int>& a,
                    const std::pair<int,int>& b) const {
        return a.second > b.second; // 按 second 小顶堆
    }
};
```

---

### 5. pair 的字典序特性

```cpp
pair<int,int> a{1,2};
pair<int,int> b{1,3};

// 比较顺序：
// 1️⃣ first
// 2️⃣ second
assert(a < b); // true
```