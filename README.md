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