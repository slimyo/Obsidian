- 子类DenseCoeffsBase（[[DenseCoeffsBase.h]]）
```cpp
                           EigenBase<Derived>
                                   ↑
                       DenseCoeffsBase<Derived, Access>
                                   ↑
                             DenseBase<Derived>
                                   ↑
                           MatrixBase<Derived>
                                   ↑1
                        PlainObjectBase<Derived>
                                   ↑
                         Matrix / Array (具体类型)

```
# 一、EigenBase 有什么作用？一句话总结

> **EigenBase 是所有“可以被赋值给 MatrixBase 的表达式类型”的共同基类，用来做函数重载区分（disambiguation）和统一接口，并为 DenseBase 的 operator=、+=、-= 做桥接。它本身几乎不做计算，是 CRTP 表达式体系的最顶层基类。**

---

#  二、EigenBase 的位置：Eigen 表达式体系的 “顶层基类”

Eigen 所有可用于赋值给矩阵的类型，都继承自：

- Matrix / Array（真正有存储）
    
- Block、Transpose、Map、DiagonalMatrix、TriangularView、CwiseBinaryOp ...（表达式）
    
- 其他稀有的 wrapper
    

这些都需要能写：

`MatrixXd A; A = someExpression;`

EigenBase 就是用来保证：

✔ `operator=(T)`、`operator+=(T)`、`operator-=(T)`  
✔ T 可以是任意表达式

---

#  三、为什么需要 EigenBase？（核心）

如果所有表达式类都继承自 MatrixBase，那么！  
会发生两个问题：

###  1. MatrixBase 不能有存储，因此实际 Matrix 不能继承它（CRTP 设计）

MatrixBase 是抽象表达式基类，没有 array[] 成员。  
Matrix 需要存储数据，所以不能直接继承 MatrixBase。

###  2. 表达式类型很多，不可能都继承 MatrixBase

例如：

- CwiseUnaryOp
    
- Block
    
- Map
    
- DiagonalMatrix
    
- Transpose
    
- Product
    

这些类型完全不同，不应该强行继承 MatrixBase。

###  因此：EigenBase 作为“最通用表达式基类”

所有“可以变成矩阵的东西”都继承`EigenBase<Derived>`。

然后 **MatrixBase 继承 EigenBase**。

所以表达式继承链通常是：

`SomeExpression<Derived>-> EigenBase<Derived>`

或者对于真正的矩阵：

```cpp
Matrix<Scalar, Rows, Cols>     
-> PlainObjectBase<Derived>     
-> DenseBase<Derived>     
-> MatrixBase<Derived>     
-> EigenBase<Derived>
```

#  四、EigenBase 的主要功能拆解

下面我逐段解释代码。

---

# 1. CRTP：提供 derived() / const_derived()

```cpp
/** \returns a reference to the derived object */

  EIGEN_DEVICE_FUNC constexpr Derived& derived() { return *static_cast<Derived*>(this); }

  /** \returns a const reference to the derived object */

  EIGEN_DEVICE_FUNC constexpr const Derived& derived() const { return *static_cast<const Derived*>(this); }

  

  EIGEN_DEVICE_FUNC inline constexpr Derived& const_cast_derived() const {

    return *static_cast<Derived*>(const_cast<EigenBase*>(this));

  }

  EIGEN_DEVICE_FUNC inline const Derived& const_derived() const { return *static_cast<const Derived*>(this); }
```

这是 CRTP 的关键：

- EigenBase 不知道自己真正的类型（如 Matrix、Block、Transpose）
    
- 通过 CRTP，这些函数可以在基类中调用派生类的接口（rows、cols、evalTo 等）
    

---

# 2. 基础几何接口 rows(), cols(), size()

```cpp
/** \returns the number of rows. \sa cols(), RowsAtCompileTime */

  EIGEN_DEVICE_FUNC constexpr Index rows() const noexcept { return derived().rows(); }

  /** \returns the number of columns. \sa rows(), ColsAtCompileTime*/

  EIGEN_DEVICE_FUNC constexpr Index cols() const noexcept { return derived().cols(); }

  /** \returns the number of coefficients, which is rows()*cols().

   * \sa rows(), cols(), SizeAtCompileTime. */

  EIGEN_DEVICE_FUNC constexpr Index size() const noexcept { return rows() * cols(); }
```

所有表达式都必须提供 rows() 和 cols()，因此 EigenBase 提供统一入口。

这使得：

- MatrixXd
    
- `Block<MatrixXd>`
    
- Transpose<CwiseBinaryOp<...>>
    
- 一切表达式
    

都能用 `.rows()`、`.size()`。

---

# 3. evalTo/addTo/subTo (表达式求值接口)

```cpp
/** \internal Don't use it, but do the equivalent: \code dst = *this; \endcode */

  template <typename Dest>

  EIGEN_DEVICE_FUNC inline void evalTo(Dest& dst) const {

    derived().evalTo(dst);

  }

  

  /** \internal Don't use it, but do the equivalent: \code dst += *this; \endcode */

  template <typename Dest>

  EIGEN_DEVICE_FUNC inline void addTo(Dest& dst) const {

    // This is the default implementation,

    // derived class can reimplement it in a more optimized way.

    typename Dest::PlainObject res(rows(), cols());

    evalTo(res);

    dst += res;

  }

  

  /** \internal Don't use it, but do the equivalent: \code dst -= *this; \endcode */

  template <typename Dest>

  EIGEN_DEVICE_FUNC inline void subTo(Dest& dst) const {

    // This is the default implementation,

    // derived class can reimplement it in a more optimized way.

    typename Dest::PlainObject res(rows(), cols());

    evalTo(res);

    dst -= res;

  }
```

例如：

`template <typename Dest> void evalTo(Dest& dst) const {     derived().evalTo(dst); }`

这是 Eigen 表达式求值的核心：

- 任何表达式在赋值给矩阵 A 时，最终会调用 evalTo(A)
    
- 派生类可以重载 evalTo，生成更高效的特化版本
    
    - 比如求一个 Block 的拷贝
        
    - 一个 “A + B” 的求值
        
    - 一个 “transpose()” 的求值
        

---

# 4. applyThisOnTheLeft / Right

```cpp
/** \internal Don't use it, but do the equivalent: \code dst.applyOnTheRight(*this); \endcode */

  template <typename Dest>

  EIGEN_DEVICE_FUNC inline void applyThisOnTheRight(Dest& dst) const {

    // This is the default implementation,

    // derived class can reimplement it in a more optimized way.

    dst = dst * this->derived();

  }

  

  /** \internal Don't use it, but do the equivalent: \code dst.applyOnTheLeft(*this); \endcode */

  template <typename Dest>

  EIGEN_DEVICE_FUNC inline void applyThisOnTheLeft(Dest& dst) const {

    // This is the default implementation,

    // derived class can reimplement it in a more optimized way.

    dst = this->derived() * dst;

  }
```
用于 lazy 表达式的矩阵乘法合成，默认通过普通乘法实现，但某些表达式可以特化，例如：

- TriangularView
    
- DiagonalMatrix
    
- PermutationMatrix
    

它们可以更快地左乘或右乘。

---

## 5. `device()`

```cpp
template <typename Device>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE DeviceWrapper<Derived, Device> device(Device& device);

  template <typename Device>

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE DeviceWrapper<const Derived, Device> device(Device& device) const;
```


---

# 五、DenseBase 重载 = / += / -= 的核心逻辑与 EigenBase 的关系

```cpp
/***************************************************************************

 * Implementation of matrix base methods

 ***************************************************************************/

  

/** \brief Copies the generic expression \a other into *this.

 *

 * \details The expression must provide a (templated) evalTo(Derived& dst) const

 * function which does the actual job. In practice, this allows any user to write

 * its own special matrix without having to modify MatrixBase

 *

 * \returns a reference to *this.

 */

template <typename Derived>

template <typename OtherDerived>

EIGEN_DEVICE_FUNC Derived& DenseBase<Derived>::operator=(const EigenBase<OtherDerived>& other) {

  call_assignment(derived(), other.derived());

  return derived();

}

  

template <typename Derived>

template <typename OtherDerived>

EIGEN_DEVICE_FUNC Derived& DenseBase<Derived>::operator+=(const EigenBase<OtherDerived>& other) {

  call_assignment(derived(), other.derived(), internal::add_assign_op<Scalar, typename OtherDerived::Scalar>());

  return derived();

}

  

template <typename Derived>

template <typename OtherDerived>

EIGEN_DEVICE_FUNC Derived& DenseBase<Derived>::operator-=(const EigenBase<OtherDerived>& other) {

  call_assignment(derived(), other.derived(), internal::sub_assign_op<Scalar, typename OtherDerived::Scalar>());

  return derived();

}

  

}  // end namespace Eigen

```

这一部分非常关键：

```cpp
template <typename Derived> 
template <typename OtherDerived> Derived& DenseBase<Derived>::operator=(const EigenBase<OtherDerived>& other) 
{   
call_assignment(derived(), other.derived());   
return derived(); 
}
```

这是 EigenBase 最大的作用！

说明：
### operator= 接受 EigenBase 的引用，而不是 Derived

- 注：Derived对象能隐式表达为`Eigenbase<Derived>&`。

- 所有 EigenBase 派生类（即所有表达式）都能合理赋值给 DenseBase 派生类（矩阵）
    参考：
	 ```cpp
		template <typename Derived>
		struct ExprBase {
		Derived& derived();
		int size() const;
		};
		
		template <typename T>
		struct Vector : ExprBase<Vector<T>> {};
		
		template <typename TA, typename TB>
		struct AddExpr : ExprBase<AddExpr<TA,TB>> {};
		
		template <typename Other>
		Vector& operator=(const ExprBase<Other>& e);
		
		ExprBase<int> x;   // 错误，不允许
		AddExpr expr;        // 派生类
		v = expr;            // OK: expr 转换到 ExprBase<AddExpr>&
		```

- 为什么不能让 operator= 参数直接接受 Derived？
例如：

`template <typename OtherDerived> Derived& operator=(const OtherDerived& other);`

这是不行的。原因：

- 会匹配过多类型，导致二义性（重载冲突）
例如：

- MatrixBase 已经有很多 operator=
    
- 许多表达式类也会隐式转换
    
- 还要避免产生与普通拷贝构造、内置类型赋值冲突
    

如果 operator= 直接写成：

`operator=(const T& other)`

会变得失控。

- 使用 EigenBase 做“表达式家族”的限制

EigenBase 的作用很重要：

> 只有继承 EigenBase 的类型才能走这个 operator=。


---

#  六、为什么单独搞一个 EigenBase，而不是直接用 DenseBase？

理由：

### 1. Sparse、DiagonalMatrix 等不是 DenseBase

但它们可以被用于生成 dense 矩阵，例如：

`MatrixXd A = diagonalMatrix;`

因此赋值操作不能直接依赖 DenseBase。

### 2. 表达式如 CwiseBinaryOp 不是 DenseBase

它们是表达式，没有存储，自然不能继承 DenseBase。

### 3. EigenBase 是非常轻量的：不包含存储、不包含复杂逻辑

仅用于：

- CRTP 统一接口
    
- 赋值兼容性
    
- 函数重载 disambiguation
    

---

# 七 、关系图总结

### 表达式类的继承层次
```cpp
             EigenBase<Derived>  
                    ↑
----------------------------------------------
|                |               |           |
MatrixBase     Block         CwiseOp     DiagonalMatrix
   ↑
DenseBase
   ↑
PlainObjectBase
   ↑
 Matrix / Array
```

---

#  八、EigenBase 的价值总结（面试级别）

📌 **为所有表达式与矩阵提供统一的 CRTP 基类**  
📌 **让 DenseBase 的 operator= 可以接收任意表达式类型**  
📌 **避免 MatrixBase 被滥用，区分表达式基类与实际矩阵属性**  
📌 **提供 rows/cols/size 等最基础接口**  
📌 **提供 evalTo / addTo / subTo 的统一入口**  
📌 **解决重载歧义（disambiguation）问题**  
📌 **为用户自定义矩阵类型提供接入 Eigen 表达式系统的门槛**

