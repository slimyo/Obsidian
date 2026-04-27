
|**方法**​|**说明**​|
|---|---|
|**构造函数与赋值**​||
|`vector()`|默认构造函数|
|`vector(size_type n)`|构造包含 n 个默认初始化元素的向量|
|`vector(size_type n, const T& value)`|构造包含 n 个 value 副本的向量|
|`vector(InputIt first, InputIt last)`|用迭代器范围构造向量|
|`vector(const vector& other)`|拷贝构造函数|
|`vector(vector&& other)`|移动构造函数 (C++11)|
|`vector(initializer_list<T> init)`|初始化列表构造 (C++11)|
|`operator=`|赋值操作符|
|`assign(size_type n, const T& value)`|用 n 个 value 副本替换内容|
|`assign(InputIt first, InputIt last)`|用迭代器范围替换内容|
|`assign(initializer_list<T> init)`|用初始化列表替换内容 (C++11)|
|**元素访问**​||
|`at(size_type pos)`|返回 pos 处元素的引用，进行边界检查|
|`operator[](size_type pos)`|返回 pos 处元素的引用，不进行边界检查|
|`front()`|返回首元素的引用|
|`back()`|返回尾元素的引用|
|`data()`|返回指向底层数组的指针|
|**迭代器**​||
|`begin()`, `end()`|返回迭代器|
|`cbegin()`, `cend()`|返回 const 迭代器 (C++11)|
|`rbegin()`, `rend()`|返回反向迭代器|
|`crbegin()`, `crend()`|返回 const 反向迭代器 (C++11)|
|**容量**​||
|`empty()`|检查容器是否为空|
|`size()`|返回元素数量|
|`max_size()`|返回可容纳的最大元素数|
|`reserve(size_type new_cap)`|预留容量，使 capacity >= new_cap|
|`capacity()`|返回当前分配的存储容量|
|`shrink_to_fit()`|请求移除未使用的容量 (C++11)|
|**修改器**​||
|`clear()`|清空所有元素|
|`push_back(const T& value)`|在尾部插入元素（拷贝）|
|`push_back(T&& value)`|在尾部插入元素（移动）(C++11)|
|`emplace_back(Args&&... args)`|在尾部就地构造元素 (C++11)|
|`pop_back()`|移除尾部元素|
|`insert(const_iterator pos, const T& value)`|在指定位置插入元素（拷贝）|
|`insert(const_iterator pos, T&& value)`|在指定位置插入元素（移动）(C++11)|
|`insert(const_iterator pos, size_type n, const T& value)`|在指定位置插入 n 个 value 副本|
|`insert(const_iterator pos, InputIt first, InputIt last)`|在指定位置插入迭代器范围|
|`insert(const_iterator pos, initializer_list<T> init)`|在指定位置插入初始化列表 (C++11)|
|`emplace(const_iterator pos, Args&&... args)`|在指定位置就地构造元素 (C++11)|
|`erase(const_iterator pos)`|删除指定位置的元素|
|`erase(const_iterator first, const_iterator last)`|删除迭代器范围的元素|
|`swap(vector& other)`|交换两个向量的内容|
|`resize(size_type count)`|改变容器大小，新增元素默认初始化|
|`resize(size_type count, const T& value)`|改变容器大小，新增元素为 value 副本|
|**观察器**​||
|`get_allocator()`|返回关联的分配器|
|**非成员函数**​||
|`operator==`, `operator!=`|比较两个向量是否相等/不相等|
|`operator<`, `operator<=`, `operator>`, `operator>=`|按字典序比较两个向量|
|`swap(vector& lhs, vector& rhs)`|交换两个向量的内容|

### 注意：

- `vector`是动态数组，支持快速随机访问
    
- 尾部插入删除是 O(1) 摊还时间，其他位置插入删除是 O(n)
    
- 容量（capacity）和大小（size）不同，容量通常大于等于大小
    
- 当容量不足时，vector 会重新分配内存（通常是翻倍），这会使所有迭代器失效
    
- 支持随机访问，可通过下标 `[]`或 `at()`访问元素
    
- 使用 `reserve()`可预分配内存，避免多次重新分配
    
- 在 C++11 及以后，`data()`返回指向底层数组的指针