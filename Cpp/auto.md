```cpp
vector<int*> vec = {...};

for(auto i:vec):
// 1. 值拷贝（指针副本）
auto i             // int*，可修改指针本身和指向的值
const auto i       // int* const，指针本身是const（不能i=nullptr），但*i可改
auto* i            // int*，同auto i
const auto* i      // const int*，指针指向的内容是const

// 2. 引用
auto& i            // int*&，指针的引用
const auto& i      // const int*&，指针的const引用
```