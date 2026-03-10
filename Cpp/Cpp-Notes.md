### 基本概念：
```cpp
bool is_aeiou(const string &s){  // 参数是 string 类型
    return (s=='a')||(s=='e')||(s=='i')||(s=='o')||(s=='u');
    // 这里用 string 和字符 'a' 比较，类型不匹配
}

// 如果像判断字符是否等于元音
bool is_aeiou_char(const char c) {
    return c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u' ||
           c == 'A' || c == 'E' || c == 'I' || c == 'O' || c == 'U';
}
```


### 三元表达式：

```cpp
value = cond ? if_value :else_value;
```

### 按位异或（XOR）

```cpp
int a=0;
int b=4;
int c =  a^b;//按位异或
```

### 类别推导的注意
```cpp
auto f = {n[i], n[j], n[k]};
vector<int> f_ = {n[i], n[j], n[k]};
sort(f.begin(),f.end());//错误
sort(f_.begin(),f_.end());

// 更正：使用auto但要明确类型
auto f = vector<int>{n[i], n[j], n[k]};
sort(f.begin(), f.end());

```
- **类型推导问题**：`auto f = {n[i], n[j], n[k]};`这行代码中，`auto`会被推导为 `std::initializer_list<int>`类型。`std::initializer_list`是一个不可变（只读）的容器，它不提供修改其元素的方法，也没有 `begin()`和 `end()`成员函数返回可修改的迭代器。

### 迭代器的注意:

```cpp
//vector<char>& s
 auto r=s.end();//指向最后的后面一个
        r--;//------------
        for(auto l=s.begin();l<r;l++){
			//main
        }
```

- auto r = s.end();  // 问题：s.end() 指向末尾后一位，解引用 `*r `是非法操作

- 与`for(auto& name : names)`区别：
```cpp
std::list<std::string> names = {"Alice", "Bob", "Charlie"};

// 范围for：迭代的是元素本身
for(auto& name : names) {  // name是std::string&
    std::cout << name << " ";
}

// 迭代器循环：迭代的是迭代器
for(auto it = names.begin(); it != names.end(); ++it) {  // it是list<string>::iterator
    std::cout << *it << " ";  // 需要解引用
}
```

- **反向迭代器:**
```cpp
// 反向遍历 vector
for (auto rit = vec.rbegin(); rit != vec.rend(); ++rit) {
    std::cout << *rit << " ";  // 输出: 5 4 3 2 1
}

// 使用范围for循环（C++11起）
std::vector<int> reversed(vec.rbegin(), vec.rend());
// 或直接反向处理
for (int value : std::vector<int>(vec.rbegin(), vec.rend())) {
    // 反向处理元素
}
```

- **使用迭代器插入后，原迭代器失效**
```cpp
std::string str = "ABCD";
auto it = str.begin() + 2;  // 指向 'C'

// ❌ 错误：插入后it失效，不要使用
str.insert(it, 'X');
// it 现在无效，不能使用
// std::cout << *it;  // 未定义行为

// ✅ 正确：使用返回值
it = str.insert(it, 'X');  // 返回插入位置的迭代器
// it 现在指向新插入的 'X'
// 继续使用it是安全的
++it;  // 现在指向原来的 'C'
```

## 完整的迭代器类型对比

|迭代器类型|声明方式|能否修改元素|能否移动迭代器|
|---|---|---|---|
|正向迭代器|`auto it = s.begin();`|✓|✓|
|常量正向迭代器|`auto cit = s.cbegin();`|✗|✓|
|反向迭代器|`auto rit = s.rbegin();`|✓|✓|
|常量反向迭代器|`auto crit = s.crbegin();`|✗|✓|
|迭代器的常量引用|`const auto& it_ref = s.begin();`|✓|✗ (引用本身)|
|元素的常量引用|`const auto& elem = *it;`|✗ (通过该引用)|✓|

### 二进制操作：

```cpp
int num = 42;  // 二进制: 00101010
    int bits = sizeof(num) * 8;  // 获取位数（通常是32位）
    
    std::cout << "整数 " << num << " 的二进制位（从高位到低位）:\n";
    
    // 从最高位到最低位
    for (int i = bits - 1; i >= 0; i--) {
        int bit = (num >> i) & 1;  // 右移i位后与1进行与操作
        std::cout << bit;
        if (i % 8 == 0 && i != 0) std::cout << " ";  // 每8位加空格
    }
    std::cout << "\n";
```

- **负数的二进制表示：**
- **补码表示**：C++ 使用补码表示有符号整数  
- **负数的计算**：`-x = ~x + 1`（按位取反再加1）
- 公式$n\&(-n)$可以得到n的二进制表示的最低位1的位置
```cpp
由于负数是按照补码规则在计算机中存储的，−n 的二进制表示为 n 的二进制表示的每一位取反再加上 1，因此它的原理如下：
假设 n 的二进制表示为 (a10⋯0) 
​
  进行按位与运算，高位全部变为 0，最低位的 1 以及之后的所有 0 不变，这样我们就获取了 n 二进制表示的最低位的 1。

因此，如果 n 是正整数并且 n & (-n) = n，那么 n 就是 2 

作者：力扣官方题解
链接：https://leetcode.cn/problems/power-of-two/solutions/796201/2de-mi-by-leetcode-solution-rny3/

```

### 容器类：

-  **关于`vector:`**
	- 初始化：
	- 想要一个 `n`行、`m`列 的二维向量，并且所有元素初始化为 `0`，可以这样写：
```cpp
int n = 3, m = 4;
std::vector<std::vector<int>> matrix(n, std::vector<int>(m));
```

- **关于set与map**：
	- 统计**出现**用set
		- erase和insert
	- 统计**次数**用map
		- map的自动插入key
			- 访问`map[key]`时，会自动增加这个key
			- 使用find操作防止意外增加不需要的key
		- map的size是计算key对的数量，不会对key的value值计数
		```cpp
		// ❌ 危险：自动插入
		int score1 = scores["Charlie"];  // 自动插入{"Charlie", 0}
		cout << scores.size() << endl;   // 输出: 3
		
		// ✅ 方法1：使用find（推荐）
		auto it = scores.find("David");
		if (it != scores.end()) {
		    cout << "David的分数: " << it->second << endl;
		} else {
		    cout << "David不存在" << endl;  // 不会插入
		}
		cout << "size: " << scores.size() << endl;  // 输出: 3
		```
		- map的insert操作与emplace操作：
			- 都不会更改原来值
			- 手动插入
				```cpp
				// 插入新key，value初始化为0
				auto result = m.insert({"apple", 0});
				if (result.second) {
				    cout << "成功插入新key: apple" << endl;
				} else {
				    cout << "key已存在" << endl;
				}
				
				// insert不会修改已存在的value
				m.insert({"apple", 100});  // ❌ 不会修改，value还是0
				
				unordered_map<string, int> m;
				-----------------------------------------------------------cpp11
				// 原地构造，避免临时对象
				auto result = m.emplace("apple", 0);  // 构造pair<string, int>("apple", 0)
				cout << "插入成功: " << result.second << endl;  // true
				
				// 已存在时不会插入
				m.emplace("apple", 100);  // 不会修改
				-----------------------------------------------------------cpp17
				-----------------------------------------如果key不存在，才构造value
				m.try_emplace("apple", 0);  // 插入
				m.try_emplace("apple", 100);  // 跳过，不会修改
				
				// 可以避免不必要的value构造
				complexClass obj(100);  // 假设构造开销大
				m.try_emplace("key", std::move(obj));  // key不存在时才移动
				```
	- 关于桶数量
		- 初始化时设置的是桶的大致数量
			- `unordered_map<string, int> m(50);  // 请求约50个桶`
		- 可以手动优化桶数避免哈希冲突
		```cpp
		// 方法A：reserve（推荐）
		unordered_set<int> set_a;
		set_a.reserve(data.size());  // 预留空间，避免rehash
		set_a.insert(data.begin(), data.end());
		
		// 方法B：构造函数指定桶数
		size_t expected_size = data.size();
		unordered_set<int> set_b(expected_size);  // 指定桶数
		set_b.insert(data.begin(), data.end());
		
		// 方法C：精确控制负载因子
		unordered_set<int> set_c;
		set_c.max_load_factor(0.5);  // 更低的冲突率
		set_c.reserve(data.size());  // 需要更多桶
		set_c.insert(data.begin(), data.end());

		```

- list类
	- insert(iterator,n):表示在迭代器前面插入
	- ## 总结表格

|特性|std::list|
|---|---|
|**索引访问**​|❌ 不支持 `list[index]`|
|**删除第一个元素**​|`pop_front()`或 `erase(begin())`|
|**默认大小**​|0|
|**预分配内存**​|❌ 没有 `reserve()`|
|**高效初始化**​|`list(n, value)`或批量构造|
|**size() 复杂度**​|O(1) (C++11起)|
|**中间插入删除**​|✅ O(1)|
|**内存连续性**​|❌ 不连续|
