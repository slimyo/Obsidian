`list`api:

| **方法**​                                                                              | **说明**​                              |
| ------------------------------------------------------------------------------------ | ------------------------------------ |
| **构造函数与赋值**​                                                                         |                                      |
| `list()`                                                                             | 默认构造函数                               |
| `list(size_type n)`                                                                  | 构造包含 n 个默认初始化元素的列表                   |
| `list(size_type n, const T& value)`                                                  | 构造包含 n 个 value 副本的列表                 |
| `list(InputIt first, InputIt last)`                                                  | 用迭代器范围构造列表                           |
| `list(const list& other)`                                                            | 拷贝构造函数                               |
| `list(list&& other)`                                                                 | 移动构造函数 (C++11)                       |
| `list(initializer_list<T> init)`                                                     | 初始化列表构造 (C++11)                      |
| `operator=`                                                                          | 赋值操作符                                |
| `assign(size_type n, const T& value)`                                                | 用 n 个 value 副本替换内容                   |
| `assign(InputIt first, InputIt last)`                                                | 用迭代器范围替换内容                           |
| `assign(initializer_list<T> init)`                                                   | 用初始化列表替换内容 (C++11)                   |
| **元素访问**​                                                                            |                                      |
| `front()`                                                                            | 返回首元素的引用                             |
| `back()`                                                                             | 返回尾元素的引用                             |
| **迭代器**​                                                                             |                                      |
| `begin()`, `end()`                                                                   | 返回迭代器                                |
| `cbegin()`, `cend()`                                                                 | 返回 const 迭代器 (C++11)                 |
| `rbegin()`, `rend()`                                                                 | 返回反向迭代器                              |
| `crbegin()`, `crend()`                                                               | 返回 const 反向迭代器 (C++11)               |
| **容量**​                                                                              |                                      |
| `empty()`                                                                            | 检查容器是否为空                             |
| `size()`                                                                             | 返回元素数量                               |
| `max_size()`                                                                         | 返回可容纳的最大元素数                          |
| **修改器**​                                                                             |                                      |
| `clear()`                                                                            | 清空所有元素                               |
| `push_back(const T& value)`                                                          | 在尾部插入元素（拷贝）                          |
| `push_back(T&& value)`                                                               | 在尾部插入元素（移动）(C++11)                   |
| `emplace_back(Args&&... args)`                                                       | 在尾部就地构造元素 (C++11)                    |
| `pop_back()`                                                                         | 移除尾部元素                               |
| `push_front(const T& value)`                                                         | 在头部插入元素（拷贝）                          |
| `push_front(T&& value)`                                                              | 在头部插入元素（移动）(C++11)                   |
| `emplace_front(Args&&... args)`                                                      | 在头部就地构造元素 (C++11)                    |
| `pop_front()`                                                                        | 移除头部元素                               |
| `insert(const_iterator pos, const T& value)`                                         | 在指定位置插入元素（拷贝）                        |
| `insert(const_iterator pos, T&& value)`                                              | 在指定位置插入元素（移动）(C++11)                 |
| `insert(const_iterator pos, size_type n, const T& value)`                            | 在指定位置插入 n 个 value 副本                 |
| `insert(const_iterator pos, InputIt first, InputIt last)`                            | 在指定位置插入迭代器范围                         |
| `insert(const_iterator pos, initializer_list<T> init)`                               | 在指定位置插入初始化列表 (C++11)                 |
| `emplace(const_iterator pos, Args&&... args)`                                        | 在指定位置就地构造元素 (C++11)                  |
| `erase(const_iterator pos)`                                                          | 删除指定位置的元素                            |
| `erase(const_iterator first, const_iterator last)`                                   | 删除迭代器范围的元素                           |
| `swap(list& other)`                                                                  | 交换两个列表的内容                            |
| `resize(size_type count)`                                                            | 改变容器大小，新增元素默认初始化                     |
| `resize(size_type count, const T& value)`                                            | 改变容器大小，新增元素为 value 副本                |
| **操作**​                                                                              |                                      |
| `splice(const_iterator pos, list& other)`                                            | 从 other 移动所有元素到 pos 前                |
| `splice(const_iterator pos, list&& other)`                                           | 从 other 移动所有元素到 pos 前 (C++11)        |
| `splice(const_iterator pos, list& other, const_iterator it)`                         | 从 other 移动单个元素 it 到 pos 前            |
| `splice(const_iterator pos, list&& other, const_iterator it)`                        | 从 other 移动单个元素 it 到 pos 前 (C++11)    |
| `splice(const_iterator pos, list& other, const_iterator first, const_iterator last)` | 从 other 移动元素范围 [first, last) 到 pos 前 |
| `remove(const T& value)`                                                             | 删除所有值等于 value 的元素                    |
| `remove_if(Predicate p)`                                                             | 删除所有满足谓词 p 的元素                       |
| `reverse()`                                                                          | 反转容器中元素顺序                            |
| `unique()`                                                                           | 删除连续重复的元素（使用 == 比较）                  |
| `unique(BinaryPredicate p)`                                                          | 删除连续满足谓词 p 的元素                       |
| `sort()`                                                                             | 对元素进行排序（使用 < 比较）                     |
| `sort(Compare comp)`                                                                 | 用比较函数 comp 排序                        |
| `merge(list& other)`                                                                 | 合并两个已排序列表（使用 < 比较）                   |
| `merge(list&& other)`                                                                | 合并两个已排序列表（使用 < 比较）(C++11)            |
| `merge(list& other, Compare comp)`                                                   | 用比较函数 comp 合并两个已排序列表                 |
| `merge(list&& other, Compare comp)`                                                  | 用比较函数 comp 合并两个已排序列表 (C++11)         |
| **观察器**​                                                                             |                                      |
| `get_allocator()`                                                                    | 返回关联的分配器                             |
| **非成员函数**​                                                                           |                                      |
| `operator==`, `operator!=`                                                           | 比较两个列表是否相等/不相等                       |
| `operator<`, `operator<=`, `operator>`, `operator>=`                                 | 按字典序比较两个列表                           |
| `swap(list& lhs, list& rhs)`                                                         | 交换两个列表的内容                            |
### 注意：

- `list`是双向链表，支持在任意位置高效插入删除
    
- 插入和删除不会使迭代器失效（被删除元素的迭代器除外）
    
- 不支持随机访问，没有 `operator[]`和 `at()`方法
    
- 大部分操作的时间复杂度为 O(1) 或 O(n)，具体取决于操作类型