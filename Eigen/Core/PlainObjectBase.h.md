
## 关系结构如图：
```swift
              DenseBase
                 ↑
    internal::dense_xpr_base<Derived>::type
                 ↑
          PlainObjectBase<Derived>
                 ↑
   ---------------------------------
   |                               |
Matrix<Scalar,Rows,Cols,Options>   Array<...>

```
- 通过中间结构dense_xpr_base:[[Xpr_helper.h]]进行表达式分发，选择：
	- MatrixBase类[[MatrixBase.h]]
	- ArrayBase类[[ArrayBase.h]]
	之一作为基类继承，提供对外API接口。
# **typedef & 基本属性（traits 取出来的各种信息）**

代码开头一大串 typedef、enum：
```cpp
enum { Options = internal::traits<Derived>::Options }; using Base = internal::dense_xpr_base<Derived>::type; using Scalar = internal::traits<Derived>::Scalar; 
... 
using MapType = Eigen::Map<Derived, Unaligned>;
```


**作用：**

- 从 `traits<Derived>` 中抽取
    
    - 静态行列数
        
    - Scalar 类型
        
    - 是否 row-major
        
    - 对齐需求
        
- “继承”表达式基类 `Base`（通常是 `MatrixBase<Derived>`）
    

---

# ② **核心：m_storage（真正的数据）**

```cpp
DenseStorage<Scalar, MaxSize, RowsAtCompileTime, ColsAtCompileTime, Options> m_storage;
```

它负责：

- data 指针
    
- rows, cols
    
- resize
    
- swap
    

所有 element 的访问最终都是：

`m_storage.data()[ index ]`

PlainObjectBase 不持有任何其他数据。

---

# ③ **系数访问（不经过 evaluator，最快路径）**

例如：
```cpp
const Scalar& coeff(Index rowId, Index colId) const {     if (Flags & RowMajorBit)         return m_storage.data()[colId + rowId * m_storage.cols()];     else         return m_storage.data()[rowId + colId * m_storage.rows()]; }`
```


这段代码绕过 evaluator，直接地址计算，所以非常快。

Eigen 的普通表达式使用 evaluator，而 PlainObjectBase 是“真实矩阵”，可以直接访问内存。

---

# ④ **resize / conservativeResize 核心逻辑**

### resize：
```cpp
`m_storage.resize(size, rows, cols);`
```


如果开启初始化选项，会写 0 或 NaN。

关键点：

- 如果新 size == 旧 size，不重新分配
    
- 如果任何静态维度不变，则检查静态维度
    
- 可能重建数组（重分配），旧数据丢弃
    

### conservativeResize：

在 `internal::conservative_resize_like_impl` 中实现：

- 如果满足 row-major & 仅改变行数，或者 col-major & 仅改变列数  
    → 可就地重新分配（风险小，性能高）
    
- 否则  
    → 必须创建临时 `tmp`，复制公共部分，然后 swap
    

---

# ⑤ **赋值逻辑（最关键的主体：表达式求值路径）**

### operator= 分为三类：

#### 1) 直接赋值 PlainObjectBase → PlainObjectBase

`operator=(const PlainObjectBase& other) {     return _set(other); }`

#### 2) 赋值表达式（MatrixBase）

`operator=(const EigenBase<OtherDerived>& other)`

首先：

`_resize_to_match(other)`

然后：

`Base::operator=(other.derived())`

由 `MatrixBase` 层处理表达式求值。

---

### 关键内部函数

#### `_set()`

带 aliasing 检查的 assignment：

`internal::call_assignment(this->derived(), other.derived());`

#### `_set_noalias()`

用于新建对象时的 lazy assignment：

`internal::call_assignment_no_alias(...)`

这是 Eigen 表达式模板的**最终落地点（sink）**。

---

# ⑥ **构造函数：多种初始化路径**

例如：

- 指定尺寸：`PlainObjectBase(Index size, Index rows, Index cols)`
    
- initializer_list 构造（支持 `{{1,2},{3,4}}`）
    
- 从表达式初始化
    
- 从 raw pointer 初始化
    

这些构造函数全部初始化 m_storage 然后执行 `_set_noalias(...)`。

---

# ⑦ Map 系列

PlainObjectBase 提供大量静态函数方便构造 `Map`：

`Map(data) Map(data, rows, cols) MapAligned(data) Map with stride`

作用：

- 把“外部数组”包装成 Eigen 对象
    
- 不拷贝数据
    

PlainObjectBase 是传统矩阵类型，Map 是“矩阵视图”。

---

# ⑧ swap 优化

RowMajor、ColMajor，以及是否 dynamic size 决定是否能：

`swap pointers`

在 dynamic-size 的矩阵之间，仅通过交换 `m_storage.data()` 和尺寸即可，避免复制。

---

#  **最终总结（最重要的 10 行）**

PlainObjectBase 主要实现：

1. **储存**：m_storage 管理 data + rows/cols。
    
2. **访问**：coeff()/coeffRef() 直接 raw pointer 计算。
    
3. **尺寸管理**：resize / conservativeResize。
    
4. **表达式求值最终收敛点**：operator= → _set / call_assignment。
    
5. **对齐**：对齐内存分配。
    
6. **Map**：构建不拷贝的矩阵视图。
    
7. **swap**：智能地交换内部数据。
    
8. **构造**：raw pointer、initializer_list、表达式都能初始化。
    
9. **安全检查**：MaxRows/MaxCols 静态检查。
    
10. **继承结构**：作为 Matrix / Array 的真正底层容器。