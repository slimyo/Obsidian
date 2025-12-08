```cpp
template <typename Derived>
class MatrixBase : public DenseBase<Derived> {...
```

- 主要数据存放于DenseStorage类([[DenseStorage.h]])
- 提供给中间分发结构dense_xpr_base([[Xpr_helper.h]])
- 作为PlainObjectBase类([[PlainObjectBase.h]])的基类
