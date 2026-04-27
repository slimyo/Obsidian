
###  `unordered_set` api:
| **方法**                         | **说明**                         |
| ------------------------------ | ------------------------------ |
| **构造函数与赋值**​                   |                                |
| `unordered_set()`              | 默认构造函数                         |
| `unordered_set(init_list)`     | 初始化列表构造                        |
| `unordered_set(begin, end)`    | 迭代器范围构造                        |
| `unordered_set(other)`         | 拷贝构造                           |
| `unordered_set(other, alloc)`  | 使用分配器拷贝构造                      |
| `operator=`                    | 赋值操作                           |
| **容量相关**​                      |                                |
| `empty()`                      | 判断集合是否为空                       |
| `size()`                       | 返回元素数量                         |
| `max_size()`                   | 返回可容纳的最大元素数                    |
| **查找与访问**​                     |                                |
| `find(key)`                    | 返回指向元素的迭代器，未找到则返回 `end()`      |
| `count(key)`                   | 返回匹配元素的数量（0 或 1）               |
| `contains(key)`(C++20)         | 返回是否存在该元素（bool）                |
| **插入操作**​                      |                                |
| `insert(value)`                | 插入元素，返回 pair<iterator, bool>   |
| `insert(iterator hint, value)` | 在位置提示后插入                       |
| `insert(begin, end)`           | 插入迭代器范围                        |
| `insert(init_list)`            | 插入初始化列表                        |
| `emplace(args...)`             | 原地构造插入，返回 pair<iterator, bool> |
| `emplace_hint(hint, args...)`  | 在位置提示后原地构造插入                   |
| **删除操作**​                      |                                |
| `erase(key)`                   | 删除指定键，返回删除的元素数量（0 或 1）         |
| `erase(iterator)`              | 删除迭代器指向的元素                     |
| `erase(begin, end)`            | 删除迭代器范围                        |
| `clear()`                      | 清空所有元素                         |
| **桶操作**​                       |                                |
| `bucket_count()`               | 返回桶的数量                         |
| `max_bucket_count()`           | 返回最大桶数                         |
| `bucket(key)`                  | 返回键所在的桶索引                      |
| `bucket_size(n)`               | 返回第 n 个桶中的元素数量                 |
| `load_factor()`                | 返回当前负载因子（元素数/桶数）               |
| `max_load_factor()`            | 获取/设置最大负载因子（可读写）               |
| `rehash(n)`                    | 将桶数量设置为至少 n                    |
| `reserve(n)`                   | 为至少 n 个元素预留空间                  |
| **哈希与比较器**​                    |                                |
| `hash_function()`              | 返回哈希函数对象                       |
| `key_eq()`                     | 返回键相等比较函数对象                    |
| **迭代器**​                       |                                |
| `begin()`, `end()`             | 返回迭代器                          |
| `cbegin()`, `cend()`           | 返回 const 迭代器                   |
| `begin(n)`, `end(n)`           | 返回第 n 个桶的迭代器                   |
| `cbegin(n)`, `cend(n)`         | 返回第 n 个桶的 const 迭代器            |
| **分配器**​                       |                                |
| `get_allocator()`              | 返回分配器对象                        |
| **特殊操作**​                      |                                |
| `swap(other)`                  | 交换两个 unordered_set 的内容         |
| `merge(other)`(C++17)          | 从其他容器合并元素（提取插入）                |