# BlockImpl_dense类

- 有两种情况
## 1.继承自dense_xpr_base([[Xpr_helper.h]])

```cpp
/** \internal Internal implementation of dense Blocks in the general case. */

template <typename XprType, int BlockRows, int BlockCols, bool InnerPanel, bool HasDirectAccess>

class BlockImpl_dense : public internal::dense_xpr_base<Block<XprType, BlockRows, BlockCols, InnerPanel>>::type {

  typedef Block<XprType, BlockRows, BlockCols, InnerPanel> BlockType;

  typedef typename internal::ref_selector<XprType>::non_const_type XprTypeNested;

  

 public:

  typedef typename internal::dense_xpr_base<BlockType>::type Base;

  EIGEN_DENSE_PUBLIC_INTERFACE(BlockType)

  EIGEN_INHERIT_ASSIGNMENT_OPERATORS(BlockImpl_dense)

  

  // class InnerIterator; // FIXME apparently never used

  

  /** Column or Row constructor

   */

  EIGEN_DEVICE_FUNC inline BlockImpl_dense(XprType& xpr, Index i)

      : m_xpr(xpr),

        // It is a row if and only if BlockRows==1 and BlockCols==XprType::ColsAtCompileTime,

        // and it is a column if and only if BlockRows==XprType::RowsAtCompileTime and BlockCols==1,

        // all other cases are invalid.

        // The case a 1x1 matrix seems ambiguous, but the result is the same anyway.

        m_startRow((BlockRows == 1) && (BlockCols == XprType::ColsAtCompileTime) ? i : 0),

        m_startCol((BlockRows == XprType::RowsAtCompileTime) && (BlockCols == 1) ? i : 0),

        m_blockRows(BlockRows == 1 ? 1 : xpr.rows()),

        m_blockCols(BlockCols == 1 ? 1 : xpr.cols()) {}

  

  /** Fixed-size constructor

   */

  EIGEN_DEVICE_FUNC inline BlockImpl_dense(XprType& xpr, Index startRow, Index startCol)

      : m_xpr(xpr), m_startRow(startRow), m_startCol(startCol), m_blockRows(BlockRows), m_blockCols(BlockCols) {}

  

  /** Dynamic-size constructor

   */

  EIGEN_DEVICE_FUNC inline BlockImpl_dense(XprType& xpr, Index startRow, Index startCol, Index blockRows,

                                           Index blockCols)

      : m_xpr(xpr), m_startRow(startRow), m_startCol(startCol), m_blockRows(blockRows), m_blockCols(blockCols) {}

  

  EIGEN_DEVICE_FUNC inline Index rows() const { return m_blockRows.value(); }

  EIGEN_DEVICE_FUNC inline Index cols() const { return m_blockCols.value(); }

  

  EIGEN_DEVICE_FUNC inline Scalar& coeffRef(Index rowId, Index colId) {

    EIGEN_STATIC_ASSERT_LVALUE(XprType)

    return m_xpr.coeffRef(rowId + m_startRow.value(), colId + m_startCol.value());

  }

  

  EIGEN_DEVICE_FUNC inline const Scalar& coeffRef(Index rowId, Index colId) const {

    return m_xpr.derived().coeffRef(rowId + m_startRow.value(), colId + m_startCol.value());

  }

  

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE const CoeffReturnType coeff(Index rowId, Index colId) const {

    return m_xpr.coeff(rowId + m_startRow.value(), colId + m_startCol.value());

  }

  

  EIGEN_DEVICE_FUNC inline Scalar& coeffRef(Index index) {

    EIGEN_STATIC_ASSERT_LVALUE(XprType)

    return m_xpr.coeffRef(m_startRow.value() + (RowsAtCompileTime == 1 ? 0 : index),

                          m_startCol.value() + (RowsAtCompileTime == 1 ? index : 0));

  }

  

  EIGEN_DEVICE_FUNC inline const Scalar& coeffRef(Index index) const {

    return m_xpr.coeffRef(m_startRow.value() + (RowsAtCompileTime == 1 ? 0 : index),

                          m_startCol.value() + (RowsAtCompileTime == 1 ? index : 0));

  }

  

  EIGEN_DEVICE_FUNC inline const CoeffReturnType coeff(Index index) const {

    return m_xpr.coeff(m_startRow.value() + (RowsAtCompileTime == 1 ? 0 : index),

                       m_startCol.value() + (RowsAtCompileTime == 1 ? index : 0));

  }

  

  template <int LoadMode>

  EIGEN_DEVICE_FUNC inline PacketScalar packet(Index rowId, Index colId) const {

    return m_xpr.template packet<Unaligned>(rowId + m_startRow.value(), colId + m_startCol.value());

  }

  

  template <int LoadMode>

  EIGEN_DEVICE_FUNC inline void writePacket(Index rowId, Index colId, const PacketScalar& val) {

    m_xpr.template writePacket<Unaligned>(rowId + m_startRow.value(), colId + m_startCol.value(), val);

  }

  

  template <int LoadMode>

  EIGEN_DEVICE_FUNC inline PacketScalar packet(Index index) const {

    return m_xpr.template packet<Unaligned>(m_startRow.value() + (RowsAtCompileTime == 1 ? 0 : index),

                                            m_startCol.value() + (RowsAtCompileTime == 1 ? index : 0));

  }

  

  template <int LoadMode>

  EIGEN_DEVICE_FUNC inline void writePacket(Index index, const PacketScalar& val) {

    m_xpr.template writePacket<Unaligned>(m_startRow.value() + (RowsAtCompileTime == 1 ? 0 : index),

                                          m_startCol.value() + (RowsAtCompileTime == 1 ? index : 0), val);

  }

  

#ifdef EIGEN_PARSED_BY_DOXYGEN

  /** \sa MapBase::data() */

  EIGEN_DEVICE_FUNC constexpr const Scalar* data() const;

  EIGEN_DEVICE_FUNC inline Index innerStride() const;

  EIGEN_DEVICE_FUNC inline Index outerStride() const;

#endif

  

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE const internal::remove_all_t<XprTypeNested>& nestedExpression() const {

    return m_xpr;

  }

  

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE XprType& nestedExpression() { return m_xpr; }

  

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr StorageIndex startRow() const noexcept { return m_startRow.value(); }

  

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr StorageIndex startCol() const noexcept { return m_startCol.value(); }

  

 protected:

  XprTypeNested m_xpr;

  const internal::variable_if_dynamic<StorageIndex, (XprType::RowsAtCompileTime == 1 && BlockRows == 1) ? 0 : Dynamic>

      m_startRow;

  const internal::variable_if_dynamic<StorageIndex, (XprType::ColsAtCompileTime == 1 && BlockCols == 1) ? 0 : Dynamic>

      m_startCol;

  const internal::variable_if_dynamic<StorageIndex, RowsAtCompileTime> m_blockRows;

  const internal::variable_if_dynamic<StorageIndex, ColsAtCompileTime> m_blockCols;

};
```


## 2.继承自MapDense([[MapBase.h]])
```cpp
/** \internal Internal implementation of dense Blocks in the direct access case.*/

template <typename XprType, int BlockRows, int BlockCols, bool InnerPanel>

class BlockImpl_dense<XprType, BlockRows, BlockCols, InnerPanel, true>

    : public MapBase<Block<XprType, BlockRows, BlockCols, InnerPanel>> {

  typedef Block<XprType, BlockRows, BlockCols, InnerPanel> BlockType;

  typedef typename internal::ref_selector<XprType>::non_const_type XprTypeNested;

  enum { XprTypeIsRowMajor = (int(traits<XprType>::Flags) & RowMajorBit) != 0 };

  

  /** \internal Returns base+offset (unless base is null, in which case returns null).

   * Adding an offset to nullptr is undefined behavior, so we must avoid it.

   */

  template <typename Scalar>

  EIGEN_DEVICE_FUNC constexpr EIGEN_ALWAYS_INLINE static Scalar* add_to_nullable_pointer(Scalar* base, Index offset) {

    return base != nullptr ? base + offset : nullptr;

  }

  

 public:

  typedef MapBase<BlockType> Base;

  EIGEN_DENSE_PUBLIC_INTERFACE(BlockType)

  EIGEN_INHERIT_ASSIGNMENT_OPERATORS(BlockImpl_dense)

  

  /** Column or Row constructor

   */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE BlockImpl_dense(XprType& xpr, Index i)

      : Base((BlockRows == 0 || BlockCols == 0)

                 ? nullptr

                 : add_to_nullable_pointer(

                       xpr.data(),

                       i * (((BlockRows == 1) && (BlockCols == XprType::ColsAtCompileTime) && (!XprTypeIsRowMajor)) ||

                                    ((BlockRows == XprType::RowsAtCompileTime) && (BlockCols == 1) &&

                                     (XprTypeIsRowMajor))

                                ? xpr.innerStride()

                                : xpr.outerStride())),

             BlockRows == 1 ? 1 : xpr.rows(), BlockCols == 1 ? 1 : xpr.cols()),

        m_xpr(xpr),

        m_startRow((BlockRows == 1) && (BlockCols == XprType::ColsAtCompileTime) ? i : 0),

        m_startCol((BlockRows == XprType::RowsAtCompileTime) && (BlockCols == 1) ? i : 0) {

    init();

  }

  

  /** Fixed-size constructor

   */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE BlockImpl_dense(XprType& xpr, Index startRow, Index startCol)

      : Base((BlockRows == 0 || BlockCols == 0)

                 ? nullptr

                 : add_to_nullable_pointer(xpr.data(),

                                           xpr.innerStride() * (XprTypeIsRowMajor ? startCol : startRow) +

                                               xpr.outerStride() * (XprTypeIsRowMajor ? startRow : startCol))),

        m_xpr(xpr),

        m_startRow(startRow),

        m_startCol(startCol) {

    init();

  }

  

  /** Dynamic-size constructor

   */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE BlockImpl_dense(XprType& xpr, Index startRow, Index startCol, Index blockRows,

                                                        Index blockCols)

      : Base((blockRows == 0 || blockCols == 0)

                 ? nullptr

                 : add_to_nullable_pointer(xpr.data(),

                                           xpr.innerStride() * (XprTypeIsRowMajor ? startCol : startRow) +

                                               xpr.outerStride() * (XprTypeIsRowMajor ? startRow : startCol)),

             blockRows, blockCols),

        m_xpr(xpr),

        m_startRow(startRow),

        m_startCol(startCol) {

    init();

  }

  

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE const internal::remove_all_t<XprTypeNested>& nestedExpression() const noexcept {

    return m_xpr;

  }

  

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE XprType& nestedExpression() { return m_xpr; }

  

  /** \sa MapBase::innerStride() */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr Index innerStride() const noexcept {

    return internal::traits<BlockType>::HasSameStorageOrderAsXprType ? m_xpr.innerStride() : m_xpr.outerStride();

  }

  

  /** \sa MapBase::outerStride() */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr Index outerStride() const noexcept {

    return internal::traits<BlockType>::HasSameStorageOrderAsXprType ? m_xpr.outerStride() : m_xpr.innerStride();

  }

  

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr StorageIndex startRow() const noexcept { return m_startRow.value(); }

  

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr StorageIndex startCol() const noexcept { return m_startCol.value(); }

  

#ifndef __SUNPRO_CC

  // FIXME sunstudio is not friendly with the above friend...

  // META-FIXME there is no 'friend' keyword around here. Is this obsolete?

 protected:

#endif

  

#ifndef EIGEN_PARSED_BY_DOXYGEN

  /** \internal used by allowAligned() */

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE BlockImpl_dense(XprType& xpr, const Scalar* data, Index blockRows,

                                                        Index blockCols)

      : Base(data, blockRows, blockCols), m_xpr(xpr) {

    init();

  }

#endif

  

 protected:

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE void init() {

    m_outerStride =

        internal::traits<BlockType>::HasSameStorageOrderAsXprType ? m_xpr.outerStride() : m_xpr.innerStride();

  }

  

  XprTypeNested m_xpr;

  const internal::variable_if_dynamic<StorageIndex, (XprType::RowsAtCompileTime == 1 && BlockRows == 1) ? 0 : Dynamic>

      m_startRow;

  const internal::variable_if_dynamic<StorageIndex, (XprType::ColsAtCompileTime == 1 && BlockCols == 1) ? 0 : Dynamic>

      m_startCol;

  Index m_outerStride;

};
```


# BlockImpl类


- 继承自BlockImpl_dense([[#^220ea8]])

# Block类
- 继承自BlockImpl([[#^8f3c5c]])