
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