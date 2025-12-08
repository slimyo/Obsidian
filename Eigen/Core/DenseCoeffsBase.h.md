- 继承自父类EigenBase([[EigenBase.h]])
- 子类：DenseBase([[DenseBase.h]])

# **1. DenseCoeffsBase 的核心意义**

**DenseCoeffsBase 是 Eigen 中所有 "可访问系数的 dense 表达式" 的底层基类。**

它是 Eigen 的表达式模板体系的重要组成部分，为如下类型服务：

- `Matrix<T>` / `Array<T>`
    
- `Map<>`
    
- `Block<>`
    
- `Transpose<>`
    
- 任意表达式如 `A + B`、`A.array() * 2` 等
    

**作用：统一定义“如何读写一个表达式中的单个元素”。**

它负责：

|访问方式|说明|
|---|---|
|`(i, j)`|普通二维访问|
|`[i]`|线性访问（向量或 LinearAccessBit）|
|`x(), y(), z(), w()`|特殊访问|
|`packet(i,j)`|SIMD vector packet 读取|
|`coeff()`|无断言版快速访问|
|`coeffRef()`|写访问版本（仅 WriteAccessors 支持）|

并根据派生类特征（Flags) 决定：

- 返回值是 `Scalar` 还是 `const Scalar&`
    
- 是否能写入（WriteAccessBit）
    
- 是否有直接内存访问（DirectAccessBit）
    
- 是否允许 LinearAccess
    
- 是否内存连续，从而支持 SIMD
    

---

# **2. DenseCoeffsBase 的 4 种 specialization**

```txt
DenseCoeffsBase<Derived, ReadOnlyAccessors>
DenseCoeffsBase<Derived, WriteAccessors>
DenseCoeffsBase<Derived, DirectAccessors>
DenseCoeffsBase<Derived, DirectWriteAccessors>
```

它们层层继承：

```
DirectWriteAccessors
	 ↑ 
DirectAccessors
	 ↑ 
WriteAccessors
	 ↑ 
ReadOnlyAccessors
```


**注意：只有 flags 告诉你派生类是否允许某种访问，不是 C++ 类型系统本身限制！**

例如 `Map<const Matrix>`：

- 不能写 → 没有 WriteAccessBit → 不允许 coeffRef()
    
- 但可以直接访问内存 → 有 DirectAccessBit → coeff() 返回 `const Scalar&`
    

---
# **3.`DenseCoeffsBase<Derived, ReadOnlyAccessors>`:**

>使用 Base 的接口（CRTP）

```cpp
typedef EigenBase<Derived> Base; 
using Base::rows; 
using Base::cols; 
using Base::size; 
using Base::derived;
```

Base 中只有最基础的：
- rows()
- cols()
- size()
- derived()

DenseCoeffsBase 负责扩展成更丰富的 coefficient API。

## **① CoeffReturnType（非常重要）**

这是本文件最难懂也最重要的 typedef：

```cpp
typedef std::conditional_t<
    bool(internal::traits<Derived>::Flags & (LvalueBit | DirectAccessBit)),
    const Scalar&,        // 有真实内存，且可引用
    std::conditional_t<internal::is_arithmetic<Scalar>::value,
                       Scalar,     // 标量：返回值拷贝
                       const Scalar>>  // 其它类型：返回 const Scalar
> CoeffReturnType;

// Explanation for this CoeffReturnType typedef.
// - This is the return type of the coeff() method.
// - The LvalueBit means exactly that we can offer a coeffRef() method, which means exactly that we can get references
// to coeffs, which means exactly that we can have coeff() return a const reference (as opposed to returning a value).
// - The DirectAccessBit means exactly that the underlying data of coefficients can be directly accessed as a plain
// strided array, which means exactly that the underlying data of coefficients does exist in memory, which means
// exactly that the coefficients is const-referencable, which means exactly that we can have coeff() return a const
// reference. For example, Map<const Matrix> have DirectAccessBit but not LvalueBit, so that Map<const Matrix>.coeff()
// does points to a const Scalar& which exists in memory, while does not allow coeffRef() as it would not provide a
// lvalue. Notice that DirectAccessBit and LvalueBit are mutually orthogonal.
// - The is_arithmetic check is required since "const int", "const double", etc. will cause warnings on some systems
// while the declaration of "const T", where T is a non arithmetic type does not. Always returning "const Scalar&" is
// not possible, since the underlying expressions might not offer a valid address the reference could be referring to.
```

它解决一个核心问题：

> 某些表达式是**没有真实内存**的，比如 `(A+B)`  
> 因此不能返回 `Scalar&` 或 `const Scalar&` → 因为没有能引用的地址！

所以：

|情况|返回类型|
|---|---|
|有直接内存 或 是 lvalue（Matrix、Map、Block）|`const Scalar&`|
|表达式类型（如 AddExpr）无真实存储|`Scalar`（按值返回）|
|标量是类类型（如 `std::complex`）|`const Scalar`|

这是 Eigen 高性能的关键：**能引用就引用，不能引用就复制。**

---
## **② rowIndexByOuterInner / colIndexByOuterInner

```cpp
EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE Index rowIndexByOuterInner(Index outer, Index inner) const {
    return int(Derived::RowsAtCompileTime) == 1   ? 0
           : int(Derived::ColsAtCompileTime) == 1 ? inner
           : int(Derived::Flags) & RowMajorBit    ? outer
                                                  : inner;
  }
  
  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE Index colIndexByOuterInner(Index outer, Index inner) const {
    return int(Derived::ColsAtCompileTime) == 1   ? 0
           : int(Derived::RowsAtCompileTime) == 1 ? inner
           : int(Derived::Flags) & RowMajorBit    ? inner
                                                  : outer;
  }
```
这是高级内容，Eigen 的统一索引系统。

用途：表达式内部可能按 “外、内” 索引访问（如行迭代器、列迭代器，或共轭块表达式），需要将 Outer/Inner 映射到真实 row/col。

规则：

```cpp
rowIndexByOuterInner:
    if RowVector (rows=1)          → row=0
    else if ColVector (cols=1)     → row=inner
    else if RowMajor               → row=outer
    else                           → row=inner

```

colIndex 类似。

用途：表达式统一接口，不需要知道 rowmajor/colmajor。

---

## ③ coeff(i,j) 的真正执行：`evaluator<Derived>`

```cpp
  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr CoeffReturnType coeff(Index row, Index col) const {

    eigen_internal_assert(row >= 0 && row < rows() && col >= 0 && col < cols());

    return internal::evaluator<Derived>(derived()).coeff(row, col);

  }
```

```cpp
return internal::evaluator<Derived>(derived()).coeff(row, col);
```

Eigen 3.4 的 evaluator 机制意义巨大：

> 所有表达式最终都由 evaluator 执行底层实际访问/计算。  
> DenseCoeffsBase 只是统一入口，不负责执行。

举例：

- 对于 MatrixXd，evaluator 直接返回 `data()[row*stride+col]`
    
- 对于 Block，evaluator 计算偏移
    
- 对于 Transpose，evaluator 交换 row/col
    
- 对于 A+B，evaluator 调用 A.evaluator + B.evaluator
    
### evaluator 是什么？

它是一个结构：

```cpp
//todo
```

Eigen 的计算是 evaluator 链 + 内联展开完成的。

---

## ④`operator()(i,j)` = 带断言的 `coeff(i,j)`

```cpp
/** \returns the coefficient at given the given row and column.
   *
   * \sa operator()(Index,Index), operator[](Index)
   */
  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr CoeffReturnType operator()(Index row, Index col) const {
    eigen_assert(row >= 0 && row < rows() && col >= 0 && col < cols());
    return coeff(row, col);
  }
```

```cpp
return coeff(row, col);
```

与 coeff(i,j) 的差别：
- coeff(i,j) 在 debug（internal debug）下断言
- operator() 在普通 debug 下断言
    
## **coeff() 与 operator() 的关系**

### `coeff()`

- 无 assertion 版本（若启用 internal debug 则会触发）
- 内部通过 evaluator 调用：

```cpp
return internal::evaluator<Derived>(derived()).coeff(row, col);
```

#### `operator()(i,j)`
- 带断言版本：
```cpp
return coeff(row,col);
```
读取路径：
```cpp
operator() → coeff → evaluator<Expr>.coeff(row,col)
```

**核心：访问最终转发到 evaluator。**
为什么 evaluator？
因为：

- `MatrixEvaluator` 直接从内存读
- `AddEvaluator` 返回加法计算结果
- `TransposeEvaluator` 做索引转换
- `BlockEvaluator` 做偏移计算

评估策略全部封装在 evaluator 中。

---
## ⑤单参数 `coeff(index) / operator[] / operator()(index)`

```cpp
/** Short version: don't use this function, use
   * \link operator[](Index) const \endlink instead.
   *
   * Long version: this function is similar to
   * \link operator[](Index) const \endlink, but without the assertion.
   * Use this for limiting the performance cost of debugging code when doing
   * repeated coefficient access. Only use this when it is guaranteed that the
   * parameter \a index is in range.
   *
   * If EIGEN_INTERNAL_DEBUGGING is defined, an assertion will be made, making this
   * function equivalent to \link operator[](Index) const \endlink.
   *
   * \sa operator[](Index) const, coeffRef(Index), coeff(Index,Index) const
   */
  
  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr CoeffReturnType coeff(Index index) const {
    EIGEN_STATIC_ASSERT(internal::evaluator<Derived>::Flags & LinearAccessBit,            THIS_COEFFICIENT_ACCESSOR_TAKING_ONE_ACCESS_IS_ONLY_FOR_EXPRESSIONS_ALLOWING_LINEAR_ACCESS)
    eigen_internal_assert(index >= 0 && index < size());
    return internal::evaluator<Derived>(derived()).coeff(index);
  }

  /** \returns the coefficient at given index.
   *
   * This method is allowed only for vector expressions, and for matrix expressions having the LinearAccessBit.
   *
   * \sa operator[](Index), operator()(Index,Index) const, x() const, y() const,
   * z() const, w() const
   */
  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr CoeffReturnType operator[](Index index) const {
    EIGEN_STATIC_ASSERT(Derived::IsVectorAtCompileTime,                      THE_BRACKET_OPERATOR_IS_ONLY_FOR_VECTORS__USE_THE_PARENTHESIS_OPERATOR_INSTEAD)
    eigen_assert(index >= 0 && index < size());
    return coeff(index);
  }

  /** \returns the coefficient at given index.
   *
   * This is synonymous to operator[](Index) const.
   *
   * This method is allowed only for vector expressions, and for matrix expressions having the LinearAccessBit.
   *
   * \sa operator[](Index), operator()(Index,Index) const, x() const, y() const,
   * z() const, w() const
   */
  
  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr CoeffReturnType operator()(Index index) const {
    eigen_assert(index >= 0 && index < size());
    return coeff(index);
  }
```

这些函数只允许：
- Vector
- 或支持 LinearAccessBit 的矩阵
    

LinearAccessBit 表示：

> 数据可沿一条线性方向连续访问（矩阵也可以）。

Eigen 保留两个版本：对性能敏感代码使用 coeff，普通用户用 operator()。

---

## ⑥x(), y(), z(), w() —— GLSL 风格的访问

```cpp
/** equivalent to operator[](0).  */
  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr CoeffReturnType x() const { return (*this)[0]; }

  /** equivalent to operator[](1).  */
  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr CoeffReturnType y() const {
    EIGEN_STATIC_ASSERT(Derived::SizeAtCompileTime == -1 || Derived::SizeAtCompileTime >= 2, OUT_OF_RANGE_ACCESS);
    return (*this)[1];
  }

  /** equivalent to operator[](2).  */
  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr CoeffReturnType z() const {
    EIGEN_STATIC_ASSERT(Derived::SizeAtCompileTime == -1 || Derived::SizeAtCompileTime >= 3, OUT_OF_RANGE_ACCESS);
    return (*this)[2];
  }

  /** equivalent to operator[](3).  */
  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr CoeffReturnType w() const {
    EIGEN_STATIC_ASSERT(Derived::SizeAtCompileTime == -1 || Derived::SizeAtCompileTime >= 4, OUT_OF_RANGE_ACCESS);
    return (*this)[3];
  }
```

允许：
- Vector2f → x(), y()
- Vector3f → x(), y(), z()
- Vector4f → x(), y(), z(), w()

用于图形向量。

---

## ⑦packet() —— SIMD 加载接口

```cpp
/** \internal
   * \returns the packet of coefficients starting at the given row and column. It is your responsibility
   * to ensure that a packet really starts there. This method is only available on expressions having the
   * PacketAccessBit.
   *
   * The \a LoadMode parameter may have the value \a #Aligned or \a #Unaligned. Its effect is to select
   * the appropriate vectorization instruction. Aligned access is faster, but is only possible for packets
   * starting at an address which is a multiple of the packet size.
   */

  template <int LoadMode>
  EIGEN_STRONG_INLINE PacketReturnType packet(Index row, Index col) const {
    typedef typename internal::packet_traits<Scalar>::type DefaultPacketType;
    eigen_internal_assert(row >= 0 && row < rows() && col >= 0 && col < cols());
    return internal::evaluator<Derived>(derived()).template packet<LoadMode, DefaultPacketType>(row, col);
  }

  /** \internal */
  template <int LoadMode>
  EIGEN_STRONG_INLINE PacketReturnType packetByOuterInner(Index outer, Index inner) const {
    return packet<LoadMode>(rowIndexByOuterInner(outer, inner), colIndexByOuterInner(outer, inner));
  }

  /** \internal
   * \returns the packet of coefficients starting at the given index. It is your responsibility
   * to ensure that a packet really starts there. This method is only available on expressions having the
   * PacketAccessBit and the LinearAccessBit.
   *
   * The \a LoadMode parameter may have the value \a #Aligned or \a #Unaligned. Its effect is to select
   * the appropriate vectorization instruction. Aligned access is faster, but is only possible for packets
   * starting at an address which is a multiple of the packet size.
   */

  template <int LoadMode>
  EIGEN_STRONG_INLINE PacketReturnType packet(Index index) const {
    EIGEN_STATIC_ASSERT(internal::evaluator<Derived>::Flags & LinearAccessBit,              THIS_COEFFICIENT_ACCESSOR_TAKING_ONE_ACCESS_IS_ONLY_FOR_EXPRESSIONS_ALLOWING_LINEAR_ACCESS)
    typedef typename internal::packet_traits<Scalar>::type DefaultPacketType;
    eigen_internal_assert(index >= 0 && index < size());
    return internal::evaluator<Derived>(derived()).template packet<LoadMode, DefaultPacketType>(index);
  }
```


`template<int LoadMode> PacketReturnType packet(i,j) const;`

- LoadMode: Aligned / Unaligned
    
- evaluator 完成实际 vector load
    

PacketReturnType 定义为：

`add_const_on_value_if_arithmetic<PacketScalar>::type`

> 如果是 arithmetic，vector 包需要 const；如果是非 arithmetic，需要值。

---

## **⑧ 保护区：防止 DenseBase using 声明失败**

最后一段：

```cpp
protected:
  // explanation: DenseBase is doing "using ..." on the methods from DenseCoeffsBase.
  // But some methods are only available in the DirectAccess case.
  // So we add dummy methods here with these names, so that "using... " doesn't fail.
  // It's not private so that the child class DenseBase can access them, and it's not public
  // either since it's an implementation detail, so has to be protected.
  void coeffRef();
  void coeffRefByOuterInner();
  void writePacket();
  void writePacketByOuterInner();
  void copyCoeff();
  void copyCoeffByOuterInner();
  void copyPacket();
  void copyPacketByOuterInner();
  void stride();
  void innerStride();
  void outerStride();
  void rowStride();
  void colStride();
```

这是一个 **trick**：
- DenseBase 使用 `using` 把子类成员导入
- 但是某些子类（ReadOnlyAccessors）没有写访问函数
- 为了保证 using 不报错，这里提供 dummy 声明

这些函数不会被调用，只是为了语言机制。

---

#  **DenseCoeffsBase<Derived, WriteAccessors> 与 ReadOnlyAccessors 的本质区别**

Eigen 将“读取系数”和“写入系数”的逻辑拆成两个层级：

|功能|实现于|
|---|---|
|**只读访问（coeff()、operator()(…) const）**|DenseCoeffsBase<Derived, ReadOnlyAccessors>|
|**可写访问（coeffRef(), operator()(…) 非 const）**|DenseCoeffsBase<Derived, WriteAccessors>|

且写访问类继承了读访问类：

`class DenseCoeffsBase<Derived, WriteAccessors>     : public DenseCoeffsBase<Derived, ReadOnlyAccessors>`

也就是说：

✔ 写访问 = 读访问 + 写功能  
❌ 读访问 ≠ 写访问

这样区分的原因如下。

---

# 🔍 **1. 为什么要将 ReadOnly 与 Write Accessors 分成两个类？**

## **理由：表达式模板中，很多表达式是不可写的**

例如：

- `A + B`（临时表达式，没有内存）
- `CwiseUnaryOp`（逐元素操作，结果也不存储）
- `Block<const Matrix>`（视图对 const 对象）

这些表达式：

- 可以读取数据（因为 evaluator 能算）
- **不能修改数据**（因为它们没有真实存储，或存储不可写）
    

因此：

|类型|是否能写|
|---|---|
|Matrix|✔|
|Map<double*>|✔|
|Map<const double*>|❌|
|A + B|❌|
|A.row(i)|✔（取决于 A 是否可写）|
|C.block(x,y)|✔或❌|
|B.transpose()|❌（转置表达式没有写通路）|

所以必须为“是否可写”建立编译期选择。

---

# 🔍 **2. 两个类做的事情不同**

## **(1) ReadOnlyAccessors 负责：**

- 返回值类型 `CoeffReturnType`
- 各种 const 读取接口：
    - `coeff(row, col) const`
    - `operator()(row, col) const`
    - `operator[](index) const`
    - `packet(...) const`
- 多维访问计算（rowIndexByOuterInner）
- 安全检查（eigen_assert）
    
它不提供任何“修改”功能。

---

## **(2) WriteAccessors 提供：**

主要增加：
###  **1. coeffRef(row, col)** ——返回可修改引用

`Scalar& coeffRef(Index row, Index col)`
它调用 evaluator：
`evaluator<Derived>(derived()).coeffRef(row, col)`
这是写操作的核心。

###  **2. 可写 operator()(row, col)**

`Scalar& operator()(Index row, Index col)`

###  **3. 可写 `operator[](index)`**

只能用于向量表达式：

`Scalar& operator[](Index index)`

###  **4. vector 组件接口 x(), y(), z(), w() 的可写版本**

例如：

`v.x() = 0; v.y() = 5;`

###  **5. 写 packet（SIMD vector）函数**

（虽然你的代码片段中写 packet 的是 protected dummy，真正的实现来自 Derived 的 evaluator）

---

# 🔍 **3. WriteAccessors 类为何不重复 ReadOnlyAccessors 的代码？**

因为它是：

```cpp
class DenseCoeffsBase<Derived, WriteAccessors> : public DenseCoeffsBase<Derived, ReadOnlyAccessors>
```

写访问继承全部读访问函数：  
✔ `coeff()`  
✔ `operator()(row,col) const`  
✔ `operator[] const`  
✔ `packet() const`

只新增：

- `coeffRef()`
- 非 const operator()
- 非 `const operator[]`
- 写 vector 成员（x y z w）
    

---

# 🔍 **4. 为什么基于 accessor tag 进行分派？**

templated class：
`template<typename Derived, int Accessors> class DenseCoeffsBase;`
Accessors 根据 traits 选择：

|对象类型|Accessors|
|---|---|
|Matrix|WriteAccessors|
|Map<double*>|WriteAccessors|
|Map<const double*>|ReadOnlyAccessors|
|A + B|ReadOnlyAccessors|
|Block<const Matrix>|ReadOnlyAccessors|
|Transpose|ReadOnlyAccessors|

因此 **编译期就能阻止对无法写的表达式做写访问**。
如你试图写：
`(A+B)(0,0) = 5;`
编译报错，因为 `(A+B)` 只有 ReadOnlyAccessors。

---

# 🔍 **5. 对比总结（非常重要）**

## **ReadOnlyAccessors 做的事情**

| 功能                  | 是否包含 |
| ------------------- | ---- |
| 读取 coeff()          | ✔    |
| operator()(…) const | ✔    |
| `operator[] const`  | ✔    |
| evaluator 读         | ✔    |
| 写引用 coeffRef        | ❌    |
| 写 operator()( )     | ❌    |
| 写 `operator[]`      | ❌    |
| SIMD 写 packet       | ❌    |

---

## **WriteAccessors 额外增加的功能**

| 功能                          |ReadOnly|Write|
|---|---|---|
| coeffRef(row,col)           |❌|✔|
| operator()(row,col) 非 const |❌|✔|
| `operator[](index)`非 const  |❌|✔|
| x() y() z() w() 非 const     |❌|✔|
| packet 写接口                  |❌|✔|

**本质区别：是否拥有“可修改引用”的路径。**

---

# 🔍 **6. 为什么 Eigen 不直接把读写功能都放在一个类里？**

如果合并成一个类：
- 临时表达式（A+B）
- const Map
- 视图表达式（block, transpose 等）

也会继承写方法 → 调用时会失败（没有 evaluator 写入口），产生编译期或运行时错误。
因此设计成两层：
✔ **读写类只给具有真实可写存储的表达式**  
✔ **其余表达式只继承只读类**

这是**表达式模板中最关键的安全机制之一**。

---

# 📌 **结论（简明）**

`DenseCoeffsBase<Derived, WriteAccessors>` 与只读版本不同之处是：

1. **提供了所有“可写入口”函数**（coeffRef/operator() 非 const 等）。
2. **继承了所有只读函数**，避免重复代码。
3. **通过 Accessor Tag 编译期控制哪些类可写、哪些不可写**。
4. 这是表达式模板中保证 const-correctness 与写安全性的关键。


---
#  1. DenseCoeffsBase 的几种“访问类型”究竟是什么？

Eigen 将访问权限设计成一组**“标签（Accessors Tags）”**：

| 标签                       | 含义                                        |
| ------------------------ | ----------------------------------------- |
| **ReadOnlyAccessors**    | 只能读元素（提供 const operator()）                |
| **WriteAccessors**       | 可读可写（提供非 const operator()、`operator[]` 等） |
| **DirectAccessors**      | 允许直接访问底层连续内存的 stride 信息（读取）               |
| **DirectWriteAccessors** | 同上，但允许写入底层内存                              |

一般矩阵（MatrixXf）最终继承的是 **DirectWriteAccessors**。  
视图（Block, Map, Transpose 的产物等）通常继承 **非 Direct** 或 **ReadOnly 版本**。

---

#  2. WriteAccessors 与 ReadOnlyAccessors 的区别
WriteAccessors 比 ReadOnlyAccessors 多：

- coeffRef(row,col)
- operator()(row,col) 非 const
- `operator[](i)` 非 const（仅 vector）
- x() y() z() w() 的写版本
    

它们本质上都是 **经过 evaluator 的引用包装**，用于写表达式结果。

---

# 3. DirectAccessors / DirectWriteAccessors 和非 Direct 版本有什么重大区别？

**👉 DirectAccessors 允许对数据存储布局进行“直接访问”**

即允许获取：

- innerStride()
- outerStride()
- rowStride()
- colStride()
    

这些 stride 对象是 **指针偏移**。

### 非 Direct 情况的限制：

ReadOnlyAccessors 或 WriteAccessors 的表达式 **不保证有固定的 stride**，例如：
- `A.transpose()` → 行列交换，内存 stride 变化
- `Block` 可能是非连续子片
- `Map` 用户给定 stride
- `CwiseUnaryOp` 不具备固定 stride
- 任意组合表达式（A+B, A*B 等）都没有连续存储

这些表达式无法提供：
`pointer + i * innerStride + j * outerStride`
因此 **没有 DirectAccessors 标签**。

---

#  4. 那么 DirectAccessors / DirectWriteAccessors 主要用于谁？

如下对象通常具备 Direct：
### ✔ 固定连续存储的 PlainObject（如 MatrixXd、ArrayXXf）

它们的 stride 是固定的：
- column-major: innerStride = 1, outerStride = rows
- row-major: innerStride = cols, outerStride = 1
### ✔ `Map<MatrixType>`

如果 Map 指定了 stride，则 stride 可直接取出。

---

#  5. DirectAccessors 实际解决的问题是什么？

### 它允许对底层内存进行 pointer-level 优化

例：

`for(i) {   p[i * outerStride + j * innerStride] = ... }`

Eigen 内部 BLAS 优化、copy kernel 优化、部分 evaluator，都依赖 stride 信息来避免使用 operator()。

**DirectAccessors 让表达式变“可线性迭代”，性能可以提升 5~30%。**

---

#  6. DenseCoeffsBase<Derived, DirectAccessors> 的功能总结

它**增加了可直接访问底层布局的方法**：
`innerStride() outerStride() rowStride() colStride()`
但仍然只有 **读取能力**，因为继承的是 ReadOnlyAccessors。
属于：
`可以读 + 可以知道 stride + 不保证能写`
例如：一些只读视图或 const 的 Map。

---

#  7. DenseCoeffsBase<Derived, DirectWriteAccessors> 的功能总结

继承自 WriteAccessors → 具备写入能力  
并增加 stride 能力 → 具备 direct 能力

属于：
`读写 + stride 访问 + 可线性上优化`
通常是：
- MatrixXf, VectorXd（绝大多数普通矩阵）
- 可写 Map（Map<T*>）
- 线性优化对象

---

# 🧠 8. 举例对比（最能说明问题）

### ⭐ A 是个普通 MatrixXd

它继承 DirectWriteAccessors：
`A.innerStride() == 1 A.outerStride() == A.rows()`
你可以做 pointer loop：
`double* p = A.data(); for(i) for(j)   p[i*A.innerStride() + j*A.outerStride()] = ...`

---

### ⭐ B = A.transpose()
B 是一个表达式，它 **不是连续存储**：
- 无 direct 访问
- 无 stride
- 不允许取底层指针 + 偏移
因此是：
`DenseCoeffsBase<Derived, ReadOnlyAccessors>`
访问只能通过 evaluator 做跳跃计算。

---
### ⭐ C = A.block(0,0,3,3)
有时连续，有时不连续，因此：
`不提供 DirectAccessors`

---

### ⭐ Map<double*> v(data, n)
用户提供 stride = 1 → 连续  
因此允许 DirectWriteAccessors：
`v.innerStride() == 1 v.outerStride() == 1`

---
#  9. 总体上，这四类 Accessors 的区别一句话概括

| Accessor                 | 写？  | 有无 stride？ | 用途                        |
| ------------------------ | --- | ---------- | ------------------------- |
| **ReadOnlyAccessors**    | ❌   | ❌          | 只能读，表达式结果或只读视图            |
| **WriteAccessors**       | ✔   | ❌          | 可写，但没有 stride（表达式写入）      |
| **DirectAccessors**      | ❌   | ✔          | 可读，能查 stride（连续或 Map 视图）  |
| **DirectWriteAccessors** | ✔   | ✔          | 可写且可查 stride（普通矩阵、可写 Map） |

---

#  10. Eigen 为什么要区分这么细？

因为：

- **表达式模板不是实存储** → 不能提供 stride
- **视图可能不连续** → stride 不固定
- **性能优化需要 stride**
- **写操作安全性不同**

这四种标签让 Eigen 在编译期选择正确的 evaluator 优化路径。

---

#  特殊模块

```cpp
namespace internal {

template <int Alignment, typename Derived, bool JustReturnZero>
struct first_aligned_impl {
  static constexpr Index run(const Derived&) noexcept { return 0; }
};

  
template <int Alignment, typename Derived>
struct first_aligned_impl<Alignment, Derived, false> {
  static inline Index run(const Derived& m) { return internal::first_aligned<Alignment>(m.data(), m.size()); }
};

  
/** \internal \returns the index of the first element of the array stored by \a m that is properly aligned with respect
 * to \a Alignment for vectorization.
 *
 * \tparam Alignment requested alignment in Bytes.
 *
 * There is also the variant first_aligned(const Scalar*, Integer) defined in Memory.h. See it for more
 * documentation.
 */
template <int Alignment, typename Derived>
static inline Index first_aligned(const DenseBase<Derived>& m) {
  enum { ReturnZero = (int(evaluator<Derived>::Alignment) >= Alignment) || !(Derived::Flags & DirectAccessBit) };
  return first_aligned_impl<Alignment, Derived, ReturnZero>::run(m.derived());
}

  
template <typename Derived>
static inline Index first_default_aligned(const DenseBase<Derived>& m) {
  typedef typename Derived::Scalar Scalar;
  typedef typename packet_traits<Scalar>::type DefaultPacketType;
  return internal::first_aligned<int(unpacket_traits<DefaultPacketType>::alignment), Derived>(m);
}

  
template <typename Derived, bool HasDirectAccess = has_direct_access<Derived>::ret>
struct inner_stride_at_compile_time {
  enum { ret = traits<Derived>::InnerStrideAtCompileTime };
};
template <typename Derived>
struct inner_stride_at_compile_time<Derived, false> {
  enum { ret = 0 };
};

  
template <typename Derived, bool HasDirectAccess = has_direct_access<Derived>::ret>
struct outer_stride_at_compile_time {
  enum { ret = traits<Derived>::OuterStrideAtCompileTime };
};

  
template <typename Derived>
struct outer_stride_at_compile_time<Derived, false> {
  enum { ret = 0 };
};
```

它们主要负责 **两类事情**：

---

# 🧠 ① 处理 _内存对齐（alignment）_，用于 SIMD（向量化）优化

核心函数：

- `first_aligned`
    
- `first_default_aligned`
    
- `first_aligned_impl` 两个偏特化版本
    

这些用于查找某个矩阵 / 数组的存储中 **第一个满足对齐要求的元素位置**。

对齐用于 SIMD：

- SSE 需要 16 字节对齐
    
- AVX 需要 32 字节
    
- AVX-512 需要 64 字节
    

不对齐会导致：

- SIMD 加载慢
    
- 有可能违反 CPU 对齐要求（旧 CPU 下会崩溃）
    
- 无法使用 `_mm_load_ps/_mm256_load_ps`，需要较慢的 unaligned load
    

所以 Eigen 在执行向量化循环（如矩阵乘法、加法、内部循环）前，会调用：

`first_aligned<Alignment>(matrix)`

以决定从第几个元素开始能安全使用 aligned load。

---

# 🧠 ② 在编译期推断 stride（内存步长）是否可用

涉及：

- `inner_stride_at_compile_time`
    
- `outer_stride_at_compile_time`
    

结合 `has_direct_access<Derived>::ret`，用来判断：

### “这个表达式（Derived）是否能提供固定 stride？”

如果不能（比如 transpose, block, 自定义 map），结果为 0。

Stride = 内存步长，用于：

`pointer + i*outerStride + j*innerStride`

如果 stride 不固定，无法做 pointer-level 优化。

---

# 🔍 下面详细解释每个模块作用

---

# 1️⃣ first_aligned_impl —— 查找首个对齐元素的策略

`template <int Alignment, typename Derived, bool JustReturnZero> struct first_aligned_impl {   static constexpr Index run(const Derived&) noexcept { return 0; } };`

**JustReturnZero = true（编译期判断）时，直接返回 0，不查找对齐。**

意味着：

- 这个表达式天然对齐
    
- 或者它不允许 direct access（无连续存储）
    

第二个版本：

`template <int Alignment, typename Derived> struct first_aligned_impl<Alignment, Derived, false> {   static inline Index run(const Derived& m) {      return internal::first_aligned<Alignment>(m.data(), m.size());    } };`

**JustReturnZero = false** 时，需要真正扫描 `m.data()` 找对齐位置。

---

# 2️⃣ first_aligned —— 外部接口（自动决定是否搜索）

`enum {    ReturnZero = (int(evaluator<Derived>::Alignment) >= Alignment)             || !(Derived::Flags & DirectAccessBit) };`

解析：

## 🔹 ReturnZero = true 的情况

- evaluator 已经保证“起点就是 aligned”
    
- 或表达式不支持 DirectAccess（无法取 data / stride）
    

例如：

- 普通 MatrixXd（对齐的）
    
- VectorXd（对齐的）
    
- 某些 Map 对齐的
    

## 🔹 ReturnZero = false 时

需要真正计算对齐位置：

`internal::first_aligned(data, size)`

---

# 3️⃣ first_default_aligned —— 使用默认向量化对齐

默认对齐由 scalar 类型的 packet 类型决定：

`typedef typename packet_traits<Scalar>::type DefaultPacketType; unpacket_traits<DefaultPacketType>::alignment`

例如矩阵是 double → 用 AVX → 对齐 32 字节。

---

# 4️⃣ inner_stride_at_compile_time / outer_stride_at_compile_time

作用：**在编译期给出 stride（如果可能）**

### 情况 1：表达式具备 DirectAccessBit

例如 MatrixXd：

`InnerStrideAtCompileTime = 1 OuterStrideAtCompileTime = RowsAtCompileTime`

### 情况 2：表达式无 DirectAccessBit

例如：

- transpose
    
- block
    
- cwise unary op
    
- 自由组合的表达式
    

则：

`ret = 0;   // 无固定 stride`

这是非常重要的，因为它让 Eigen 能根据情况决定：

- 是否可以展开成 pointer-based loop（最快）
    
- 是否需要 evaluator(row,col)（较慢）
    

---

# 🧨 极重要：为什么这么设计？

SIMD 指令要求：

- 读写的内存必须满足特定对齐
    
- 最好是固定 stride
    
- 最好能直接根据指针做循环
    

三者不满足则：

- 不能做向量化
    
- 只能用 evaluator 的 scalar 访问（慢）
    

Eigen 的 evaluator pipeline 需要：

- 判断是否能向量化
    
- 判断从哪里开始对齐块
    
- 判断 stride 是否固定
    
- 为循环选择最佳 kernel（aligned / unaligned / scalar）
    

这些内部 helper 让 evaluator 在**编译期 + 运行期**同时选择最优执行路径。

---

# 📘 这些模块属于哪个Eigen层级？

它们属于：

`Eigen/Core/util Eigen/Core/functors Eigen/Core/Memory.h Eigen/Core/DenseCoeffsBase.h`

属于“中级底层”：  
**高于 storage、低于 evaluator、服务于 vectorization pipeline。**

---

# 🎯 总结一句话

这些 internal 模块负责：

`为 Eigen 的向量化（SIMD）和内存对齐优化提供： - 对齐起点搜索 - 对齐策略选择（是否直接0） - stride 编译期推断`

它们是 Eigen 性能的重要基石，确保：

- 连续存储 → 使用 aligned load
    
- 非连续表达式 → 自动降级为 scalar loop


---
# **附. DenseCoeffsBase 的设计哲学总结**

DenseCoeffsBase 体现了 Eigen 设计的核心理念：

## **① CRTP + traits + flags 解耦编译期信息**

派生类通过 traits 提供：
- 是否可写（WriteAccessBit）
- 是否能直接访问内存（DirectAccessBit）
- 是否线性访问（LinearAccessBit）
    
基类根据 flags 决定：
- 返回引用还是值
- 是否允许 packet / stride
→ **全部编译期优化，零运行时成本。**

---

## **② 所有真实计算推迟到 evaluator 层**

DenseCoeffsBase 只是 “访问接口”，不参与具体计算。
表达式模板体系：
`DenseCoeffsBase ← DenseBase ← MatrixBase evaluator<Expr> ← 处理真正的计算逻辑`

---

## **③ 读写访问完全分层（访问控制基于 CRTP + flags，而不是 C++ 继承权限）**

`Map<const Matrix>` 可以很好地被禁止写操作：
- 不是因为 C++ const 成员函数限制
- 也不是因为继承层次限制
- 而是因为 `traits<Derived>::Flags` 称它没有 `WriteAccessBit`

这是一种**编译期权限系统**。