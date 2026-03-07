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