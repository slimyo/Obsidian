# C++ string 核心API速查表

## 一、增（插入/追加）

|操作|语法|示例|说明|
|---|---|---|---|
|**追加字符串**​|`append(str)`|`str.append(" World");`|追加字符串|
|追加子串|`append(str, pos, n)`|`str.append(other, 6, 5);`|追加部分字符串|
|**追加运算符**​|`+=`|`str += "!";`|最常用，可追加字符或字符串|
|**追加字符**​|`push_back(ch)`|`str.push_back('!');`|只能追加单个字符|
|**插入字符串**​|`insert(pos, str)`|`str.insert(5, " ");`|指定位置插入|
|插入字符|`insert(pos, n, ch)`|`str.insert(3, 2, '-');`|插入n个相同字符|
|插入范围|`insert(it, begin, end)`|`str.insert(it, v.begin(), v.end());`|用迭代器范围插入|

## 二、删（删除）

|操作|语法|示例|说明|
|---|---|---|---|
|**删除末尾**​|`pop_back()`|`str.pop_back();`|C++11，删除最后一个字符|
|**删除子串**​|`erase(pos, n)`|`str.erase(5, 6);`|删除从pos开始的n个字符|
|删除迭代器|`erase(it)`|`str.erase(str.begin()+3);`|删除指定位置的字符|
|删除范围|`erase(begin, end)`|`str.erase(it1, it2);`|删除迭代器范围|
|**清空**​|`clear()`|`str.clear();`|删除全部内容|
|移除空白|`+算法`|见下文"实用技巧"|与算法配合使用|

## 三、查（查找/获取）

| 操作         | 语法                         | 示例                                                      | 说明                   |
| ---------- | -------------------------- | ------------------------------------------------------- | -------------------- |
| **查找字符串**​ | `find(str, pos=0)`         | `if (t.find(s) != std::string::npos) return true; // r` | 返回首次出现位置，未找到返回`npos` |
| **查找字符**​  | `find(ch, pos=0)`          | `pos = str.find('a');`                                  | 查找字符首次出现             |
| **反向查找**​  | `rfind(str, pos=npos)`     | `pos = str.rfind("abc");`                               | 从后向前查找               |
| 查找任意字符     | `find_first_of(chars)`     | `pos = str.find_first_of("aeiou");`                     | 查找集合中任意字符            |
| 查找非字符      | `find_first_not_of(chars)` | `pos = str.find_first_not_of("123");`                   | 查找非集合中字符             |
| **获取子串**​  | `substr(pos=0, n=npos)`    | `sub = str.substr(3, 5);`                               | 获取子字符串               |
| **获取字符**​  | `at(pos)`                  | `ch = str.at(3);`                                       | 安全访问，会检查边界           |
| 下标访问       | `operator[]`               | `ch = str[3];`                                          | 快速访问，不检查边界           |

## 四、改（修改/替换）

|操作|语法|示例|说明|
|---|---|---|---|
|**替换子串**​|`replace(pos, n, new_str)`|`str.replace(5, 2, "XYZ");`|替换指定位置内容|
|替换范围|`replace(begin, end, new_str)`|`str.replace(it1, it2, "new");`|用迭代器范围替换|
|赋值|`assign(str)`|`str.assign("new");`|多种重载形式|
|交换|`swap(other)`|`str1.swap(str2);`|快速交换两个字符串|
|调整大小|`resize(n, ch='\0')`|`str.resize(10, ' ');`|调整字符串长度|

## 五、容易忘记的重要API

|操作|语法|示例|为什么容易忘|
|---|---|---|---|
|**C字符串转换**​|`c_str()`|`printf("%s", str.c_str());`|与C函数交互时必需|
|**获取数据指针**​|`data()`|`write(fd, str.data(), n);`|C++17前与c_str()基本相同|
|**预留空间**​|`reserve(n)`|`str.reserve(1000);`|优化性能，但常被忽略|
|**实际容量**​|`capacity()`|`cap = str.capacity();`|查看实际分配的内存|
|**缩减容量**​|`shrink_to_fit()`|`str.shrink_to_fit();`|C++11，减少内存占用|
|**比较字符串**​|`compare(str)`|`if(str.compare("abc") == 0)`|比`==`能获取更多信息|
|**复制到C数组**​|`copy(buf, n, pos=0)`|`str.copy(buffer, 10, 0);`|手动复制到字符数组|

## 六、实用技巧（记住这些）

### 1. 去除空白

```cpp
// 去除首尾空格
str.erase(0, str.find_first_not_of(" \t\n\r"));
str.erase(str.find_last_not_of(" \t\n\r") + 1);

// 移除所有空格
str.erase(remove(str.begin(), str.end(), ' '), str.end());
```

### 2. int2string
```cpp
std::string s = "id=";
int x = 10;

s = s + std::to_string(x);
// 或
s += std::to_string(x);
```