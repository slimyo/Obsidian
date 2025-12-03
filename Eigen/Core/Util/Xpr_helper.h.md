
## dense_xpr_base结构：

- <span style="color:red">表达式类型分发（dispatch）:</span>
	-  存放（定义）两种基类（之一）：
		- MatrixBase基类[[MatrixBase.h]]或者ArrayBase基类[[ArrayBase.h]]
	- 决定PlianObjectBase类[[PlainObjectBase.h]]]存放的数据类型(基类选择器):

```cpp
  

template <typename Derived, typename XprKind = typename traits<Derived>::XprKind>

struct dense_xpr_base {

  /* dense_xpr_base should only ever be used on dense expressions, thus falling either into the MatrixXpr or into the

   * ArrayXpr cases */

};

  

template <typename Derived>

struct dense_xpr_base<Derived, MatrixXpr> {

  typedef MatrixBase<Derived> type;

};

  

template <typename Derived>

struct dense_xpr_base<Derived, ArrayXpr> {

  typedef ArrayBase<Derived> type;

};

  

template <typename Derived, typename XprKind = typename traits<Derived>::XprKind,

          typename StorageKind = typename traits<Derived>::StorageKind>

struct generic_xpr_base;

  

template <typename Derived, typename XprKind>

struct generic_xpr_base<Derived, XprKind, Dense> {

  typedef typename dense_xpr_base<Derived, XprKind>::type type;

};
```