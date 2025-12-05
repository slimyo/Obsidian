个人Obsidian笔记仓库

# 1. Eigen 的 CRTP：为什么基类需要知道 Derived？

你看到的：

`template<typename Derived> class MatrixBase : public DenseBase<Derived>`

采用 **CRTP（Curiously Recurring Template Pattern）**。

核心思想：

> 基类通过模板参数知道派生类的类型，从而在基类中写出“泛型行为”，由派生类提供具体实现。

例如：

`Derived& derived() { return *static_cast<Derived*>(this); }`

基类内部所有运算都用 `derived()` 来访问真正的派生类，这使得：

### 🔥 行为（操作）在基类

### 🔥 数据（storage）在派生类

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