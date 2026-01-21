个人Obsidian笔记仓库

# 1. Eigen 的 CRTP：

>基类用派生类Derived作为模板参数：

```cpp
template<typename Derived> class MatrixBase : public DenseBase<Derived>
```

采用 **CRTP（Curiously Recurring Template Pattern）**。

核心思想：

> 基类通过模板参数知道派生类的类型，从而在基类中写出“泛型行为”，由派生类提供具体实现。

例如：
```cpp
Derived& derived() { return *static_cast<Derived*>(this); }
```

基类内部所有运算都用 `derived()` 来访问真正的派生类，这使得：

#### 行为（操作）在基类

#### 数据（storage）在派生类

这样：

- 创建不同维度的矩阵、向量、Map、Block、CwiseUnaryOp… 时无需重新实现基本运算
    
- 任何新“表达式类型”也自动继承所有通用的操作（逐元素、加减乘、转置等）
    
- 静态多态 + 完全内联，无虚函数开销

## CRTP的典型应用

- 消除临时对象

```cpp
// 表达式模板基类
template <typename Derived>
class VectorExpression {
public:
    // 下标运算符
    double operator[](size_t i) const {
        return static_cast<const Derived&>(*this)[i];
    }
    
    // 获取大小
    size_t size() const {
        return static_cast<const Derived&>(*this).size();
    }
};

// 具体向量类
class Vector : public VectorExpression<Vector> {
    std::vector<double> data;
public:
    Vector(size_t n, double val = 0) : data(n, val) {}
    Vector(std::initializer_list<double> init) : data(init) {}
    
    double operator[](size_t i) const { return data[i]; }
    double& operator[](size_t i) { return data[i]; }
    size_t size() const { return data.size(); }
    
    // 表达式赋值 - 关键优化！
    template <typename E>
    Vector& operator=(const VectorExpression<E>& expr) {
        for (size_t i = 0; i < size(); ++i) {
            data[i] = expr[i];  // 内联展开，无临时对象！
        }
        return *this;
    }
};

// 向量加法表达式
template <typename E1, typename E2>
class VectorSum : public VectorExpression<VectorSum<E1, E2>> {
    const E1& lhs;
    const E2& rhs;
public:
    VectorSum(const E1& l, const E2& r) : lhs(l), rhs(r) {}
    
    double operator[](size_t i) const {
        return lhs[i] + rhs[i];  // 延迟计算！
    }
    
    size_t size() const { return lhs.size(); }
};

// 运算符重载
template <typename E1, typename E2>
VectorSum<E1, E2> operator+(const VectorExpression<E1>& lhs,
                            const VectorExpression<E2>& rhs) {
    return VectorSum<E1, E2>(static_cast<const E1&>(lhs),
                             static_cast<const E2&>(rhs));
}

int main() {
    Vector v1 = {1, 2, 3, 4, 5};
    Vector v2 = {5, 4, 3, 2, 1};
    Vector v3 = {2, 2, 2, 2, 2};
    
    // 关键：这里不会创建临时Vector！
    // 计算在赋值时内联展开为：v1[i] = v2[i] + v3[i];
    v1 = v2 + v3;
    
    // 更复杂的表达式也不会创建临时对象
    v1 = v1 + v2 + v3 + v1;
    
    return 0;
}
```

- 注：
	- 关键：
	> v2 + v3 (运算符+重载)
	> VectorSum<vector,vector>(表达式类对象)
	> v1 = VectorSum (类vector的运算符=重载)
	> v1\[i] = VectorExpression<VectorSum<vector,vector>>&expr\[i](多态)
	 >v1\[i] = v2\[i] + v3\[i](内联展开为)
	- 复杂情况：
```cpp
// 表达式
v1 + v2 + v3 + v1
↓
(((v1 + v2) + v3) + v1)
↓
VectorSum<VectorSum<VectorSum<Vector, Vector>, Vector>, Vector>
    // 层次结构：
    // 最外层：VectorSum<..., Vector> 表示 (... + v1)
    // 中间层：VectorSum<..., Vector> 表示 ((v1+v2) + v3)
    // 最内层：VectorSum<Vector, Vector> 表示 (v1 + v2)

// 计算过程
v1.operator=(复杂表达式)
↓
for (i...) {
    data[i] = 复杂表达式[i];
}
↓
// 完全展开为：
for (size_t i = 0; i < size(); ++i) {
    data[i] = ((v1[i] + v2[i]) + v3[i]) + v1[i];
}
// 一次循环完成所有计算！
```

## 会有标量临时变量，但这是不可避免的

`v1[i] = (v2[i] + v3[i]) * 2.0 + v1[i] / 3.0;`

在CPU层面执行时，实际上会变成：

```cpp
double temp1 = v2[i] + v3[i];    // 寄存器中的临时标量 
double temp2 = temp1 * 2.0;       // 寄存器中的临时标量 
double temp3 = v1[i] / 3.0;       // 寄存器中的临时标量 
double temp4 = temp2 + temp3;     // 寄存器中的临时标量 
v1[i] = temp4;
```

## 但这是最优的，无法进一步优化！

理由如下：
### 1. **CPU架构限制**

- 现代CPU一次只能执行一个浮点运算
    
- 每个运算都需要从寄存器中取操作数，存结果到寄存器
    
- 这些"标量临时变量"通常就在寄存器中，不是内存分配
    
### 2. **与真正"中间向量"的对比**

**传统方式（有中间向量）：**

```cpp
Vector temp1 = v2 + v3;        // 分配N个double的内存，N次赋值 
Vector temp2 = temp1 * 2.0;    // 又分配N个double的内存，N次赋值 
Vector temp3 = v1 / 3.0;       // 又分配N个double的内存，N次赋值 
Vector temp4 = temp2 + temp3;  // 又分配N个double的内存，N次赋值 
v1 = temp4;                    // 分配内存或拷贝N个元素 
// 总计：4N次内存分配 + 4N次赋值操作
```

**表达式模板：**

```cpp
for (i = 0; i < N; i++) {     // 所有计算在寄存器中完成     
v1[i] = (v2[i] + v3[i]) * 2.0 + v1[i] / 3.0; } // 总计：N次赋值操作，无额外内存分配
```

### 3. **性能差异巨大**

假设向量长度 N=1000，double 8字节：

| 方式    | 内存分配           | 内存访问      | 缓存行为 |
| ----- | -------------- | --------- | ---- |
| 传统    | 4×8KB=32KB额外内存 | 4×1000次读写 | 缓存颠簸 |
| 表达式模板 | 0KB额外内存        | 1000次写    | 缓存友好 |

## 表达式模板的真正价值

表达式模板消除的是**容器级别的中间对象**，而不是**计算过程中的寄存器临时值**。

### 消除的代价：

-  容器内存分配/释放
    
-  中间结果的内存拷贝
    
-  多次遍历数据的缓存不友好
    
-  动态内存管理的开销
    

### 保留的代价（不可避免）：

-  CPU寄存器中的中间标量值
    
-  浮点运算本身的开销

## 编译器视角

好的编译器（启用优化如 `-O2`, `-O3`）甚至可能将表达式模板的代码优化为SIMD指令：

```cpp
// 可能被优化为（SIMD）： 
for (i = 0; i < N; i += 4) {     
__m256d v2_vec = _mm256_load_pd(&v2[i]);     
__m256d v3_vec = _mm256_load_pd(&v3[i]);     
__m256d v1_vec = _mm256_load_pd(&v1[i]);          
__m256d temp = _mm256_add_pd(v2_vec, v3_vec);     
temp = _mm256_mul_pd(temp, _mm256_set1_pd(2.0));     
__m256d temp2 = _mm256_div_pd(v1_vec, _mm256_set1_pd(3.0));     
__m256d result = _mm256_add_pd(temp, temp2);          
_mm256_store_pd(&v1[i], result); 
} 
// 这里仍然有向量寄存器中的"临时变量"，但这是SIMD计算必需的
```
## 结论

表达式模板**不能消除所有临时变量**，但这是理论上的极限：

1. **物理极限**：CPU需要寄存器来存储中间计算结果
    
2. **优化的目标**：消除的是**容器对象**而非**计算过程**
    
3. **实际效果**：与手写最优循环的性能几乎相同
    

表达式模板的价值在于：

- 让 `v1 = (v2 + v3) * 2.0 + v1 / 3.0;`这样的高级语法
    
- 获得与手写 `for`循环相同的性能
    
- 避免了传统重载运算符的巨大开销
    