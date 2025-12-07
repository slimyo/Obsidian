
## 关系结构如图：
```swift
              DenseBase
                 ↑
    internal::dense_xpr_base<Derived>::type//MarixBase or ArrayBase(表达式公共接口)
                 ↑
          PlainObjectBase<Derived>//opration =
                 ↑
   ---------------------------------
   |                               |
Matrix<Scalar,Rows,Cols,Options>   Array<...>

```
- 通过中间结构dense_xpr_base:[[Xpr_helper.h]]进行表达式分发，选择：
	- MatrixBase类[[MatrixBase.h]]
	- ArrayBase类[[ArrayBase.h]]
	之一作为基类继承，提供对外API接口。
# **① typedef & 基本属性（traits 取出来的各种信息）**

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

# ③ **系数访问Coeff**


```cpp
  /** This is an overloaded version of DenseCoeffsBase<Derived,ReadOnlyAccessors>::coeff(Index,Index) const

   * provided to by-pass the creation of an evaluator of the expression, thus saving compilation efforts.

   *

   * See DenseCoeffsBase<Derived,ReadOnlyAccessors>::coeff(Index) const for details. */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr const Scalar& coeff(Index rowId, Index colId) const {

    if (Flags & RowMajorBit)

      return m_storage.data()[colId + rowId * m_storage.cols()];

    else  // column-major

      return m_storage.data()[rowId + colId * m_storage.rows()];

  }

  

  /** This is an overloaded version of DenseCoeffsBase<Derived,ReadOnlyAccessors>::coeff(Index) const

   * provided to by-pass the creation of an evaluator of the expression, thus saving compilation efforts.

   *

   * See DenseCoeffsBase<Derived,ReadOnlyAccessors>::coeff(Index) const for details. */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr const Scalar& coeff(Index index) const {

    return m_storage.data()[index];

  }

  

  /** This is an overloaded version of DenseCoeffsBase<Derived,WriteAccessors>::coeffRef(Index,Index) const

   * provided to by-pass the creation of an evaluator of the expression, thus saving compilation efforts.

   *

   * See DenseCoeffsBase<Derived,WriteAccessors>::coeffRef(Index,Index) const for details. */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr Scalar& coeffRef(Index rowId, Index colId) {

    if (Flags & RowMajorBit)

      return m_storage.data()[colId + rowId * m_storage.cols()];

    else  // column-major

      return m_storage.data()[rowId + colId * m_storage.rows()];

  }

  

  /** This is an overloaded version of DenseCoeffsBase<Derived,WriteAccessors>::coeffRef(Index) const

   * provided to by-pass the creation of an evaluator of the expression, thus saving compilation efforts.

   *

   * See DenseCoeffsBase<Derived,WriteAccessors>::coeffRef(Index) const for details. */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr Scalar& coeffRef(Index index) { return m_storage.data()[index]; }

  

  /** This is the const version of coeffRef(Index,Index) which is thus synonym of coeff(Index,Index).

   * It is provided for convenience. */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr const Scalar& coeffRef(Index rowId, Index colId) const {

    if (Flags & RowMajorBit)

      return m_storage.data()[colId + rowId * m_storage.cols()];

    else  // column-major

      return m_storage.data()[rowId + colId * m_storage.rows()];

  }

  

  /** This is the const version of coeffRef(Index) which is thus synonym of coeff(Index).

   * It is provided for convenience. */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr const Scalar& coeffRef(Index index) const {

    return m_storage.data()[index];

  }
```

> 注:
-  Eigen 默认是 Column-major（列优先存储）。矩阵元素依次存储：
```makefile
col0: a00 a10 a20 ...
col1: a01 a11 a21 ...
col2: ...

```
- 因此，data()\[row + col * rows]就是C指针正常访问方式。

Eigen 的普通表达式使用 evaluator，而 PlainObjectBase 是“真实矩阵”，可以直接访问内存。

---

# ④ packet & data

```cpp
/** \internal */

  template <int LoadMode>

  EIGEN_STRONG_INLINE PacketScalar packet(Index rowId, Index colId) const {

    return internal::ploadt<PacketScalar, LoadMode>(

        m_storage.data() + (Flags & RowMajorBit ? colId + rowId * m_storage.cols() : rowId + colId * m_storage.rows()));

  }

  

  /** \internal */

  template <int LoadMode>

  EIGEN_STRONG_INLINE PacketScalar packet(Index index) const {

    return internal::ploadt<PacketScalar, LoadMode>(m_storage.data() + index);

  }

  

  /** \internal */

  template <int StoreMode>

  EIGEN_STRONG_INLINE void writePacket(Index rowId, Index colId, const PacketScalar& val) {

    internal::pstoret<Scalar, PacketScalar, StoreMode>(

        m_storage.data() + (Flags & RowMajorBit ? colId + rowId * m_storage.cols() : rowId + colId * m_storage.rows()),

        val);

  }

  

  /** \internal */

  template <int StoreMode>

  EIGEN_STRONG_INLINE void writePacket(Index index, const PacketScalar& val) {

    internal::pstoret<Scalar, PacketScalar, StoreMode>(m_storage.data() + index, val);

  }

  

  /** \returns a const pointer to the data array of this matrix */

  EIGEN_DEVICE_FUNC constexpr const Scalar* data() const { return m_storage.data(); }

  

  /** \returns a pointer to the data array of this matrix */

  EIGEN_DEVICE_FUNC constexpr Scalar* data() { return m_storage.data(); }
```

---
# ⑤ **resize / conservativeResize 核心逻辑**


```cpp
/** Resizes \c *this to a \a rows x \a cols matrix.

   *

   * This method is intended for dynamic-size matrices, although it is legal to call it on any

   * matrix as long as fixed dimensions are left unchanged. If you only want to change the number

   * of rows and/or of columns, you can use resize(NoChange_t, Index), resize(Index, NoChange_t).

   *

   * If the current number of coefficients of \c *this exactly matches the

   * product \a rows * \a cols, then no memory allocation is performed and

   * the current values are left unchanged. In all other cases, including

   * shrinking, the data is reallocated and all previous values are lost.

   *

   * Example: \include Matrix_resize_int_int.cpp

   * Output: \verbinclude Matrix_resize_int_int.out

   *

   * \sa resize(Index) for vectors, resize(NoChange_t, Index), resize(Index, NoChange_t)

   */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr void resize(Index rows, Index cols) {

    eigen_assert(internal::check_implication(RowsAtCompileTime != Dynamic, rows == RowsAtCompileTime) &&

                 internal::check_implication(ColsAtCompileTime != Dynamic, cols == ColsAtCompileTime) &&

                 internal::check_implication(RowsAtCompileTime == Dynamic && MaxRowsAtCompileTime != Dynamic,

                                             rows <= MaxRowsAtCompileTime) &&

                 internal::check_implication(ColsAtCompileTime == Dynamic && MaxColsAtCompileTime != Dynamic,

                                             cols <= MaxColsAtCompileTime) &&

                 rows >= 0 && cols >= 0 && "Invalid sizes when resizing a matrix or array.");

#ifndef EIGEN_NO_DEBUG

    internal::check_rows_cols_for_overflow<MaxSizeAtCompileTime, MaxRowsAtCompileTime, MaxColsAtCompileTime>::run(rows,

                                                                                                                  cols);

#endif

#ifdef EIGEN_INITIALIZE_COEFFS

    Index size = rows * cols;

    bool size_changed = size != this->size();

    m_storage.resize(size, rows, cols);

    if (size_changed) EIGEN_INITIALIZE_COEFFS_IF_THAT_OPTION_IS_ENABLED

#else

    m_storage.resize(rows * cols, rows, cols);

#endif

  }

  

  /** Resizes \c *this to a vector of length \a size

   *

   * \only_for_vectors. This method does not work for

   * partially dynamic matrices when the static dimension is anything other

   * than 1. For example it will not work with Matrix<double, 2, Dynamic>.

   *

   * Example: \include Matrix_resize_int.cpp

   * Output: \verbinclude Matrix_resize_int.out

   *

   * \sa resize(Index,Index), resize(NoChange_t, Index), resize(Index, NoChange_t)

   */

  EIGEN_DEVICE_FUNC constexpr void resize(Index size) {

    EIGEN_STATIC_ASSERT_VECTOR_ONLY(PlainObjectBase)

    eigen_assert(((SizeAtCompileTime == Dynamic && (MaxSizeAtCompileTime == Dynamic || size <= MaxSizeAtCompileTime)) ||

                  SizeAtCompileTime == size) &&

                 size >= 0);

#ifdef EIGEN_INITIALIZE_COEFFS

    bool size_changed = size != this->size();

#endif

    if (RowsAtCompileTime == 1)

      m_storage.resize(size, 1, size);

    else

      m_storage.resize(size, size, 1);

#ifdef EIGEN_INITIALIZE_COEFFS

    if (size_changed) EIGEN_INITIALIZE_COEFFS_IF_THAT_OPTION_IS_ENABLED

#endif

  }

  

  /** Resizes the matrix, changing only the number of columns. For the parameter of type NoChange_t, just pass the

   * special value \c NoChange as in the example below.

   *

   * Example: \include Matrix_resize_NoChange_int.cpp

   * Output: \verbinclude Matrix_resize_NoChange_int.out

   *

   * \sa resize(Index,Index)

   */

  EIGEN_DEVICE_FUNC constexpr void resize(NoChange_t, Index cols) { resize(rows(), cols); }

  

  /** Resizes the matrix, changing only the number of rows. For the parameter of type NoChange_t, just pass the special

   * value \c NoChange as in the example below.

   *

   * Example: \include Matrix_resize_int_NoChange.cpp

   * Output: \verbinclude Matrix_resize_int_NoChange.out

   *

   * \sa resize(Index,Index)

   */

  EIGEN_DEVICE_FUNC constexpr void resize(Index rows, NoChange_t) { resize(rows, cols()); }

  

  /** Resizes \c *this to have the same dimensions as \a other.

   * Takes care of doing all the checking that's needed.

   *

   * Note that copying a row-vector into a vector (and conversely) is allowed.

   * The resizing, if any, is then done in the appropriate way so that row-vectors

   * remain row-vectors and vectors remain vectors.

   */

  template <typename OtherDerived>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE void resizeLike(const EigenBase<OtherDerived>& _other) {

    const OtherDerived& other = _other.derived();

#ifndef EIGEN_NO_DEBUG

    internal::check_rows_cols_for_overflow<MaxSizeAtCompileTime, MaxRowsAtCompileTime, MaxColsAtCompileTime>::run(

        other.rows(), other.cols());

#endif

    const Index othersize = other.rows() * other.cols();

    if (RowsAtCompileTime == 1) {

      eigen_assert(other.rows() == 1 || other.cols() == 1);

      resize(1, othersize);

    } else if (ColsAtCompileTime == 1) {

      eigen_assert(other.rows() == 1 || other.cols() == 1);

      resize(othersize, 1);

    } else

      resize(other.rows(), other.cols());

  }

  

  /** Resizes the matrix to \a rows x \a cols while leaving old values untouched.

   *

   * The method is intended for matrices of dynamic size. If you only want to change the number

   * of rows and/or of columns, you can use conservativeResize(NoChange_t, Index) or

   * conservativeResize(Index, NoChange_t).

   *

   * Matrices are resized relative to the top-left element. In case values need to be

   * appended to the matrix they will be uninitialized.

   */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE void conservativeResize(Index rows, Index cols) {

    internal::conservative_resize_like_impl<Derived>::run(*this, rows, cols);

  }

  

  /** Resizes the matrix to \a rows x \a cols while leaving old values untouched.

   *

   * As opposed to conservativeResize(Index rows, Index cols), this version leaves

   * the number of columns unchanged.

   *

   * In case the matrix is growing, new rows will be uninitialized.

   */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE void conservativeResize(Index rows, NoChange_t) {

    // Note: see the comment in conservativeResize(Index,Index)

    conservativeResize(rows, cols());

  }

  

  /** Resizes the matrix to \a rows x \a cols while leaving old values untouched.

   *

   * As opposed to conservativeResize(Index rows, Index cols), this version leaves

   * the number of rows unchanged.

   *

   * In case the matrix is growing, new columns will be uninitialized.

   */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE void conservativeResize(NoChange_t, Index cols) {

    // Note: see the comment in conservativeResize(Index,Index)

    conservativeResize(rows(), cols);

  }

  

  /** Resizes the vector to \a size while retaining old values.

   *

   * \only_for_vectors. This method does not work for

   * partially dynamic matrices when the static dimension is anything other

   * than 1. For example it will not work with Matrix<double, 2, Dynamic>.

   *

   * When values are appended, they will be uninitialized.

   */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE void conservativeResize(Index size) {

    internal::conservative_resize_like_impl<Derived>::run(*this, size);

  }

  

  /** Resizes the matrix to \a rows x \a cols of \c other, while leaving old values untouched.

   *

   * The method is intended for matrices of dynamic size. If you only want to change the number

   * of rows and/or of columns, you can use conservativeResize(NoChange_t, Index) or

   * conservativeResize(Index, NoChange_t).

   *

   * Matrices are resized relative to the top-left element. In case values need to be

   * appended to the matrix they will copied from \c other.

   */

  template <typename OtherDerived>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE void conservativeResizeLike(const DenseBase<OtherDerived>& other) {

    internal::conservative_resize_like_impl<Derived, OtherDerived>::run(*this, other);

  }
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
#### 关键内部函数conservative_resize_like_impl():

```cpp
namespace internal {

  

template <typename Derived, typename OtherDerived, bool IsVector>

struct conservative_resize_like_impl {

  static constexpr bool IsRelocatable = std::is_trivially_copyable<typename Derived::Scalar>::value;

  static void run(DenseBase<Derived>& _this, Index rows, Index cols) {

    if (_this.rows() == rows && _this.cols() == cols) return;

    EIGEN_STATIC_ASSERT_DYNAMIC_SIZE(Derived)

  

    if (IsRelocatable &&

        ((Derived::IsRowMajor && _this.cols() == cols) ||  // row-major and we change only the number of rows

         (!Derived::IsRowMajor && _this.rows() == rows)))  // column-major and we change only the number of columns

    {

#ifndef EIGEN_NO_DEBUG

      internal::check_rows_cols_for_overflow<Derived::MaxSizeAtCompileTime, Derived::MaxRowsAtCompileTime,

                                             Derived::MaxColsAtCompileTime>::run(rows, cols);

#endif

      _this.derived().m_storage.conservativeResize(rows * cols, rows, cols);

    } else {

      // The storage order does not allow us to use reallocation.

      Derived tmp(rows, cols);

      const Index common_rows = numext::mini(rows, _this.rows());

      const Index common_cols = numext::mini(cols, _this.cols());

      tmp.block(0, 0, common_rows, common_cols) = _this.block(0, 0, common_rows, common_cols);

      _this.derived().swap(tmp);

    }

  }

  

  static void run(DenseBase<Derived>& _this, const DenseBase<OtherDerived>& other) {

    if (_this.rows() == other.rows() && _this.cols() == other.cols()) return;

  

    // Note: Here is space for improvement. Basically, for conservativeResize(Index,Index),

    // neither RowsAtCompileTime or ColsAtCompileTime must be Dynamic. If only one of the

    // dimensions is dynamic, one could use either conservativeResize(Index rows, NoChange_t) or

    // conservativeResize(NoChange_t, Index cols). For these methods new static asserts like

    // EIGEN_STATIC_ASSERT_DYNAMIC_ROWS and EIGEN_STATIC_ASSERT_DYNAMIC_COLS would be good.

    EIGEN_STATIC_ASSERT_DYNAMIC_SIZE(Derived)

    EIGEN_STATIC_ASSERT_DYNAMIC_SIZE(OtherDerived)

  

    if (IsRelocatable &&

        ((Derived::IsRowMajor && _this.cols() == other.cols()) ||  // row-major and we change only the number of rows

         (!Derived::IsRowMajor &&

          _this.rows() == other.rows())))  // column-major and we change only the number of columns

    {

      const Index new_rows = other.rows() - _this.rows();

      const Index new_cols = other.cols() - _this.cols();

      _this.derived().m_storage.conservativeResize(other.size(), other.rows(), other.cols());

      if (new_rows > 0)

        _this.bottomRightCorner(new_rows, other.cols()) = other.bottomRows(new_rows);

      else if (new_cols > 0)

        _this.bottomRightCorner(other.rows(), new_cols) = other.rightCols(new_cols);

    } else {

      // The storage order does not allow us to use reallocation.

      Derived tmp(other);

      const Index common_rows = numext::mini(tmp.rows(), _this.rows());

      const Index common_cols = numext::mini(tmp.cols(), _this.cols());

      tmp.block(0, 0, common_rows, common_cols) = _this.block(0, 0, common_rows, common_cols);

      _this.derived().swap(tmp);

    }

  }

};

  

// Here, the specialization for vectors inherits from the general matrix case

// to allow calling .conservativeResize(rows,cols) on vectors.

template <typename Derived, typename OtherDerived>

struct conservative_resize_like_impl<Derived, OtherDerived, true>

    : conservative_resize_like_impl<Derived, OtherDerived, false> {

  typedef conservative_resize_like_impl<Derived, OtherDerived, false> Base;

  using Base::IsRelocatable;

  using Base::run;

  

  static void run(DenseBase<Derived>& _this, Index size) {

    const Index new_rows = Derived::RowsAtCompileTime == 1 ? 1 : size;

    const Index new_cols = Derived::RowsAtCompileTime == 1 ? size : 1;

    if (IsRelocatable)

      _this.derived().m_storage.conservativeResize(size, new_rows, new_cols);

    else

      Base::run(_this.derived(), new_rows, new_cols);

  }

  

  static void run(DenseBase<Derived>& _this, const DenseBase<OtherDerived>& other) {

    if (_this.rows() == other.rows() && _this.cols() == other.cols()) return;

  

    const Index num_new_elements = other.size() - _this.size();

  

    const Index new_rows = Derived::RowsAtCompileTime == 1 ? 1 : other.rows();

    const Index new_cols = Derived::RowsAtCompileTime == 1 ? other.cols() : 1;

    if (IsRelocatable)

      _this.derived().m_storage.conservativeResize(other.size(), new_rows, new_cols);

    else

      Base::run(_this.derived(), new_rows, new_cols);

  

    if (num_new_elements > 0) _this.tail(num_new_elements) = other.tail(num_new_elements);

  }

};
```

---
# ⑤ **赋值逻辑（最关键的主体：表达式求值路径）**

```cpp
  

  /** This is a special case of the templated operator=. Its purpose is to

   * prevent a default operator= from hiding the templated operator=.

   */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr Derived& operator=(const PlainObjectBase& other) {

    return _set(other);

  }

  

  /** \sa MatrixBase::lazyAssign() */

  template <typename OtherDerived>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE Derived& lazyAssign(const DenseBase<OtherDerived>& other) {

    _resize_to_match(other);

    return Base::lazyAssign(other.derived());

  }

  

  template <typename OtherDerived>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE Derived& operator=(const ReturnByValue<OtherDerived>& func) {

    resize(func.rows(), func.cols());

    return Base::operator=(func);

  }

  // \li \c --------一段构造函数定义

 public:

  /** \brief Copies the generic expression \a other into *this.

   * \copydetails DenseBase::operator=(const EigenBase<OtherDerived> &other)

   */

  template <typename OtherDerived>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE Derived& operator=(const EigenBase<OtherDerived>& other) {

    _resize_to_match(other);

    Base::operator=(other.derived());

    return this->derived();

  }
```

### operator= 分为三类：

#### 1) 直接赋值 PlainObjectBase → PlainObjectBase

```cpp
operator=(const PlainObjectBase& other) {     return _set(other); }
```

#### 2) 赋值表达式（MatrixBase）

```cpp
operator=(const EigenBase<OtherDerived>& other)
```

首先：

```cpp
_resize_to_match(other)
```

然后：

```cpp
Base::operator=(other.derived())
```

由 `MatrixBase` 层处理表达式求值。

---

### 关键内部函数

#### resize_to_match()

```cpp
protected:

  /** \internal Resizes *this in preparation for assigning \a other to it.

   * Takes care of doing all the checking that's needed.

   *

   * Note that copying a row-vector into a vector (and conversely) is allowed.

   * The resizing, if any, is then done in the appropriate way so that row-vectors

   * remain row-vectors and vectors remain vectors.

   */

  template <typename OtherDerived>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE void _resize_to_match(const EigenBase<OtherDerived>& other) {

#ifdef EIGEN_NO_AUTOMATIC_RESIZING

    eigen_assert((this->size() == 0 || (IsVectorAtCompileTime ? (this->size() == other.size())

                                                              : (rows() == other.rows() && cols() == other.cols()))) &&

                 "Size mismatch. Automatic resizing is disabled because EIGEN_NO_AUTOMATIC_RESIZING is defined");

    EIGEN_ONLY_USED_FOR_DEBUG(other);

#else

    resizeLike(other);

#endif

  }
```

#### `_set()`

```cpp
/**

   * \brief Copies the value of the expression \a other into \c *this with automatic resizing.

   *

   * *this might be resized to match the dimensions of \a other. If *this was a null matrix (not already initialized),

   * it will be initialized.

   *

   * Note that copying a row-vector into a vector (and conversely) is allowed.

   * The resizing, if any, is then done in the appropriate way so that row-vectors

   * remain row-vectors and vectors remain vectors.

   *

   * \sa operator=(const MatrixBase<OtherDerived>&), _set_noalias()

   *

   * \internal

   */

  // aliasing is dealt once in internal::call_assignment

  // so at this stage we have to assume aliasing... and resising has to be done later.

  template <typename OtherDerived>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr Derived& _set(const DenseBase<OtherDerived>& other) {

    internal::call_assignment(this->derived(), other.derived());

    return this->derived();

  }
```

带 aliasing 检查的 assignment：

`internal::call_assignment(this->derived(), other.derived());`

#### `_set_noalias()`

```cpp
/** \internal Like _set() but additionally makes the assumption that no aliasing effect can happen (which

   * is the case when creating a new matrix) so one can enforce lazy evaluation.

   *

   * \sa operator=(const MatrixBase<OtherDerived>&), _set()

   */

  template <typename OtherDerived>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr Derived& _set_noalias(const DenseBase<OtherDerived>& other) {

    // I don't think we need this resize call since the lazyAssign will anyways resize

    // and lazyAssign will be called by the assign selector.

    //_resize_to_match(other);

    // the 'false' below means to enforce lazy evaluation. We don't use lazyAssign() because

    // it wouldn't allow to copy a row-vector into a column-vector.

    internal::call_assignment_no_alias(this->derived(), other.derived(),

                                       internal::assign_op<Scalar, typename OtherDerived::Scalar>());

    return this->derived();

  }
```

用于新建对象时的 lazy assignment：

`internal::call_assignment_no_alias(...)`

这是 Eigen 表达式模板的**最终落地点（sink）**。

---

# ⑥ **构造函数：多种初始化路径**

```cpp
 // Prevent user from trying to instantiate PlainObjectBase objects

  // by making all its constructor protected. See bug 1074.

 protected:

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr PlainObjectBase() = default;

  /** \brief Move constructor */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr PlainObjectBase(PlainObjectBase&&) = default;

  /** \brief Move assignment operator */

  EIGEN_DEVICE_FUNC constexpr PlainObjectBase& operator=(PlainObjectBase&& other) noexcept {

    m_storage = std::move(other.m_storage);

    return *this;

  }

  

  /** Copy constructor */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr PlainObjectBase(const PlainObjectBase&) = default;

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE PlainObjectBase(Index size, Index rows, Index cols)

      : m_storage(size, rows, cols) {}

  

  /** \brief Construct a row of column vector with fixed size from an arbitrary number of coefficients.

   *

   * \only_for_vectors

   *

   * This constructor is for 1D array or vectors with more than 4 coefficients.

   *

   * \warning To construct a column (resp. row) vector of fixed length, the number of values passed to this

   * constructor must match the the fixed number of rows (resp. columns) of \c *this.

   */

  template <typename... ArgTypes>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE PlainObjectBase(const Scalar& a0, const Scalar& a1, const Scalar& a2,

                                                        const Scalar& a3, const ArgTypes&... args)

      : m_storage() {

    EIGEN_STATIC_ASSERT_VECTOR_SPECIFIC_SIZE(PlainObjectBase, sizeof...(args) + 4);

    m_storage.data()[0] = a0;

    m_storage.data()[1] = a1;

    m_storage.data()[2] = a2;

    m_storage.data()[3] = a3;

    Index i = 4;

    auto x = {(m_storage.data()[i++] = args, 0)...};

    static_cast<void>(x);

  }

  

  /** \brief Constructs a Matrix or Array and initializes it by elements given by an initializer list of initializer

   * lists

   */

  EIGEN_DEVICE_FUNC explicit constexpr EIGEN_STRONG_INLINE PlainObjectBase(

      const std::initializer_list<std::initializer_list<Scalar>>& list)

      : m_storage() {

    size_t list_size = 0;

    if (list.begin() != list.end()) {

      list_size = list.begin()->size();

    }

  

    // This is to allow syntax like VectorXi {{1, 2, 3, 4}}

    if (ColsAtCompileTime == 1 && list.size() == 1) {

      eigen_assert(list_size == static_cast<size_t>(RowsAtCompileTime) || RowsAtCompileTime == Dynamic);

      resize(list_size, ColsAtCompileTime);

      if (list.begin()->begin() != nullptr) {

        Index index = 0;

        for (const Scalar& e : *list.begin()) {

          coeffRef(index++) = e;

        }

      }

    } else {

      eigen_assert(list.size() == static_cast<size_t>(RowsAtCompileTime) || RowsAtCompileTime == Dynamic);

      eigen_assert(list_size == static_cast<size_t>(ColsAtCompileTime) || ColsAtCompileTime == Dynamic);

      resize(list.size(), list_size);

  

      Index row_index = 0;

      for (const std::initializer_list<Scalar>& row : list) {

        eigen_assert(list_size == row.size());

        Index col_index = 0;

        for (const Scalar& e : row) {

          coeffRef(row_index, col_index) = e;

          ++col_index;

        }

        ++row_index;

      }

    }

  }

  

  /** \sa PlainObjectBase::operator=(const EigenBase<OtherDerived>&) */

  template <typename OtherDerived>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE PlainObjectBase(const DenseBase<OtherDerived>& other) : m_storage() {

    resizeLike(other);

    _set_noalias(other);

  }

  

  /** \sa PlainObjectBase::operator=(const EigenBase<OtherDerived>&) */

  template <typename OtherDerived>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE PlainObjectBase(const EigenBase<OtherDerived>& other) : m_storage() {

    resizeLike(other);

    *this = other.derived();

  }

  /** \brief Copy constructor with in-place evaluation */

  template <typename OtherDerived>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE PlainObjectBase(const ReturnByValue<OtherDerived>& other) {

    // FIXME this does not automatically transpose vectors if necessary

    resize(other.rows(), other.cols());

    other.evalTo(this->derived());

  }

```
例如：

- 指定尺寸：`PlainObjectBase(Index size, Index rows, Index cols)`
    
- initializer_list 构造（支持 `{{1,2},{3,4}}`）
    
- 从表达式初始化
    
- 从 raw pointer 初始化
    

这些构造函数全部初始化 m_storage 然后执行 `_set_noalias(...)`。

---

# ⑦ Map 系列

```cpp
/** \name Map

   * These are convenience functions returning Map objects. The Map() static functions return unaligned Map objects,

   * while the AlignedMap() functions return aligned Map objects and thus should be called only with 16-byte-aligned

   * \a data pointers.

   *

   * Here is an example using strides:

   * \include Matrix_Map_stride.cpp

   * Output: \verbinclude Matrix_Map_stride.out

   *

   * \see class Map

   */

  ///@{

  static inline ConstMapType Map(const Scalar* data) { return ConstMapType(data); }

  static inline MapType Map(Scalar* data) { return MapType(data); }

  static inline ConstMapType Map(const Scalar* data, Index size) { return ConstMapType(data, size); }

  static inline MapType Map(Scalar* data, Index size) { return MapType(data, size); }

  static inline ConstMapType Map(const Scalar* data, Index rows, Index cols) { return ConstMapType(data, rows, cols); }

  static inline MapType Map(Scalar* data, Index rows, Index cols) { return MapType(data, rows, cols); }

  

  static inline ConstAlignedMapType MapAligned(const Scalar* data) { return ConstAlignedMapType(data); }

  static inline AlignedMapType MapAligned(Scalar* data) { return AlignedMapType(data); }

  static inline ConstAlignedMapType MapAligned(const Scalar* data, Index size) {

    return ConstAlignedMapType(data, size);

  }

  static inline AlignedMapType MapAligned(Scalar* data, Index size) { return AlignedMapType(data, size); }

  static inline ConstAlignedMapType MapAligned(const Scalar* data, Index rows, Index cols) {

    return ConstAlignedMapType(data, rows, cols);

  }

  static inline AlignedMapType MapAligned(Scalar* data, Index rows, Index cols) {

    return AlignedMapType(data, rows, cols);

  }

  

  template <int Outer, int Inner>

  static inline typename StridedConstMapType<Stride<Outer, Inner>>::type Map(const Scalar* data,

                                                                             const Stride<Outer, Inner>& stride) {

    return typename StridedConstMapType<Stride<Outer, Inner>>::type(data, stride);

  }

  template <int Outer, int Inner>

  static inline typename StridedMapType<Stride<Outer, Inner>>::type Map(Scalar* data,

                                                                        const Stride<Outer, Inner>& stride) {

    return typename StridedMapType<Stride<Outer, Inner>>::type(data, stride);

  }

  template <int Outer, int Inner>

  static inline typename StridedConstMapType<Stride<Outer, Inner>>::type Map(const Scalar* data, Index size,

                                                                             const Stride<Outer, Inner>& stride) {

    return typename StridedConstMapType<Stride<Outer, Inner>>::type(data, size, stride);

  }

  template <int Outer, int Inner>

  static inline typename StridedMapType<Stride<Outer, Inner>>::type Map(Scalar* data, Index size,

                                                                        const Stride<Outer, Inner>& stride) {

    return typename StridedMapType<Stride<Outer, Inner>>::type(data, size, stride);

  }

  template <int Outer, int Inner>

  static inline typename StridedConstMapType<Stride<Outer, Inner>>::type Map(const Scalar* data, Index rows, Index cols,

                                                                             const Stride<Outer, Inner>& stride) {

    return typename StridedConstMapType<Stride<Outer, Inner>>::type(data, rows, cols, stride);

  }

  template <int Outer, int Inner>

  static inline typename StridedMapType<Stride<Outer, Inner>>::type Map(Scalar* data, Index rows, Index cols,

                                                                        const Stride<Outer, Inner>& stride) {

    return typename StridedMapType<Stride<Outer, Inner>>::type(data, rows, cols, stride);

  }

  

  template <int Outer, int Inner>

  static inline typename StridedConstAlignedMapType<Stride<Outer, Inner>>::type MapAligned(

      const Scalar* data, const Stride<Outer, Inner>& stride) {

    return typename StridedConstAlignedMapType<Stride<Outer, Inner>>::type(data, stride);

  }

  template <int Outer, int Inner>

  static inline typename StridedAlignedMapType<Stride<Outer, Inner>>::type MapAligned(

      Scalar* data, const Stride<Outer, Inner>& stride) {

    return typename StridedAlignedMapType<Stride<Outer, Inner>>::type(data, stride);

  }

  template <int Outer, int Inner>

  static inline typename StridedConstAlignedMapType<Stride<Outer, Inner>>::type MapAligned(

      const Scalar* data, Index size, const Stride<Outer, Inner>& stride) {

    return typename StridedConstAlignedMapType<Stride<Outer, Inner>>::type(data, size, stride);

  }

  template <int Outer, int Inner>

  static inline typename StridedAlignedMapType<Stride<Outer, Inner>>::type MapAligned(

      Scalar* data, Index size, const Stride<Outer, Inner>& stride) {

    return typename StridedAlignedMapType<Stride<Outer, Inner>>::type(data, size, stride);

  }

  template <int Outer, int Inner>

  static inline typename StridedConstAlignedMapType<Stride<Outer, Inner>>::type MapAligned(

      const Scalar* data, Index rows, Index cols, const Stride<Outer, Inner>& stride) {

    return typename StridedConstAlignedMapType<Stride<Outer, Inner>>::type(data, rows, cols, stride);

  }

  template <int Outer, int Inner>

  static inline typename StridedAlignedMapType<Stride<Outer, Inner>>::type MapAligned(

      Scalar* data, Index rows, Index cols, const Stride<Outer, Inner>& stride) {

    return typename StridedAlignedMapType<Stride<Outer, Inner>>::type(data, rows, cols, stride);

  }

  ///@}
```
PlainObjectBase 提供大量静态函数方便构造 `Map`：

`Map(data) Map(data, rows, cols) MapAligned(data) Map with stride`

作用：

- 把“外部数组”包装成 Eigen 对象
    
- 不拷贝数据
    

PlainObjectBase 是传统矩阵类型，Map 是“矩阵视图”。

---

# ⑧ swap 优化

```cpp
public:

#ifndef EIGEN_PARSED_BY_DOXYGEN

  /** \internal

   * \brief Override DenseBase::swap() since for dynamic-sized matrices

   * of same type it is enough to swap the data pointers.

   */

  template <typename OtherDerived>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE void swap(DenseBase<OtherDerived>& other) {

    enum {SwapPointers = internal::is_same<Derived, OtherDerived>::value && Base::SizeAtCompileTime == Dynamic};

    internal::matrix_swap_impl<Derived, OtherDerived, bool(SwapPointers)>::run(this->derived(), other.derived());

  }

  

  /** \internal

   * \brief const version forwarded to DenseBase::swap

   */

  template <typename OtherDerived>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE void swap(DenseBase<OtherDerived> const& other) {

    Base::swap(other.derived());

  }

  

  enum {IsPlainObjectBase = 1};

#endif
```

RowMajor、ColMajor，以及是否 dynamic size 决定是否能：

`swap pointers`

在 dynamic-size 的矩阵之间，仅通过交换 `m_storage.data()` 和尺寸即可，避免复制。

---
#### 关键内部函数matrix_swap_impl()：
```cpp
## name space internal:
template <typename MatrixTypeA, typename MatrixTypeB, bool SwapPointers>

struct matrix_swap_impl {

  EIGEN_DEVICE_FUNC static EIGEN_STRONG_INLINE void run(MatrixTypeA& a, MatrixTypeB& b) { a.base().swap(b); }

};

  

template <typename MatrixTypeA, typename MatrixTypeB>

struct matrix_swap_impl<MatrixTypeA, MatrixTypeB, true> {

  EIGEN_DEVICE_FUNC static inline void run(MatrixTypeA& a, MatrixTypeB& b) {

    static_cast<typename MatrixTypeA::Base&>(a).m_storage.swap(static_cast<typename MatrixTypeB::Base&>(b).m_storage);

  }

};
```

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