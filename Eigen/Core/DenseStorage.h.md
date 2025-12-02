- [ ] 阅读DenseStorage.h
## 三种底层实现形式：

1. 静态矩阵：
- 形式一：
```cpp
template <typename T, int Size, int Rows, int Cols, int Options>

class DenseStorage_impl {

  plain_array<T, Size, Options> m_data;

 public:
 // \c函数实现
};
```

- 形式二：
```cpp

template <typename T, int Size, int Cols, int Options>
class DenseStorage_impl<T, Size, Dynamic, Cols, Options> {
  plain_array<T, Size, Options> m_data;
  Index m_rows = 0;

 public:
  
};
```

- 形式三：
```cpp
template <typename T, int Size, int Rows, int Options>

class DenseStorage_impl<T, Size, Rows, Dynamic, Options> {

  plain_array<T, Size, Options> m_data;

  Index m_cols = 0;

 public:

};
```
- 形式四：
```cpp
class DenseStorage_impl<T, Size, Dynamic, Dynamic, Options> {

  plain_array<T, Size, Options> m_data;

  Index m_rows = 0;

  Index m_cols = 0;

 public:
};
```
2. 零矩阵：
- 形式一：
```cpp
template <typename T, int Rows, int Cols, int Options>

class DenseStorage_impl<T, 0, Rows, Cols, Options> {

 public:

};
```
- 形式二：
```cpp
template <typename T, int Cols, int Options>

class DenseStorage_impl<T, 0, Dynamic, Cols, Options> {

  Index m_rows = 0;

 public:
};
```
- 形式三： 
```cpp
template <typename T, int Rows, int Options>

class DenseStorage_impl<T, 0, Rows, Dynamic, Options> {

  Index m_cols = 0;

 public:

};
```
- 形式四：
```cpp
template <typename T, int Options>

class DenseStorage_impl<T, 0, Dynamic, Dynamic, Options> {

  Index m_rows = 0;

  Index m_cols = 0;

 public:

};
```

3. 动态矩阵：
- 形式一：
```cpp
template <typename T, int Cols, int Options>

class DenseStorage_impl<T, Dynamic, Dynamic, Cols, Options> {

  static constexpr bool Align = (Options & DontAlign) == 0;

  T* m_data = nullptr;

  Index m_rows = 0;

 public:

  static constexpr int Size = Dynamic;

};
```
- 形式二：
```cpp
template <typename T, int Rows, int Options>

class DenseStorage_impl<T, Dynamic, Rows, Dynamic, Options> {

  static constexpr bool Align = (Options & DontAlign) == 0;

  T* m_data = nullptr;

  Index m_cols = 0;


 public:

  static constexpr int Size = Dynamic;

};
```
- 形式三：
```cpp
template <typename T, int Options>

class DenseStorage_impl<T, Dynamic, Dynamic, Dynamic, Options> {

  static constexpr bool Align = (Options & DontAlign) == 0;

  T* m_data = nullptr;

  Index m_rows = 0;

  Index m_cols = 0;

  

 public:

  static constexpr int Size = Dynamic;
};
```
- 形式四(不支持)：
```cpp
// fixed-size matrix with dynamic memory allocation not currently supported
template <typename T, int Rows, int Cols, int Options>

class DenseStorage_impl<T, Dynamic, Rows, Cols, Options> {};
```

## 静态矩阵的存放，对齐实现模板：

```cpp
template <typename T, int Size, int MatrixOrArrayOptions,

          int Alignment = (MatrixOrArrayOptions & DontAlign) ? 0 : compute_default_alignment<T, Size>::value>

struct plain_array {

  EIGEN_ALIGN_TO_BOUNDARY(Alignment) T array[Size];

#if defined(EIGEN_NO_DEBUG) || defined(EIGEN_TESTING_PLAINOBJECT_CTOR)

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr plain_array() = default;

#else

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr plain_array() {

    EIGEN_MAKE_UNALIGNED_ARRAY_ASSERT(Alignment)

    EIGEN_MAKE_STACK_ALLOCATION_ASSERT(Size * sizeof(T))

  }

#endif

};

  

template <typename T, int Size, int MatrixOrArrayOptions>

struct plain_array<T, Size, MatrixOrArrayOptions, 0> {

  // on some 32-bit platforms, stack-allocated arrays are aligned to 4 bytes, not the preferred alignment of T

  EIGEN_ALIGN_TO_BOUNDARY(alignof(T)) T array[Size];

#if defined(EIGEN_NO_DEBUG) || defined(EIGEN_TESTING_PLAINOBJECT_CTOR)

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr plain_array() = default;

#else

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr plain_array() { EIGEN_MAKE_STACK_ALLOCATION_ASSERT(Size * sizeof(T)) }

#endif

};

  

template <typename T, int Size, int Options, int Alignment>

EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr void swap_plain_array(plain_array<T, Size, Options, Alignment>& a,

                                                                      plain_array<T, Size, Options, Alignment>& b,

                                                                      Index a_size, Index b_size) {

  Index common_size = numext::mini(a_size, b_size);

  std::swap_ranges(a.array, a.array + common_size, b.array);

  if (a_size > b_size)

    smart_copy(a.array + common_size, a.array + a_size, b.array + common_size);

  else if (b_size > a_size)

    smart_copy(b.array + common_size, b.array + b_size, a.array + common_size);

}
```

### smart_copy()函数实现细节：

- 在Core/Util/memory.h中：
	[[Memory.h]] : smart_copy()
## DenseStrorage类对外实现：

```cpp
template <typename T, int Size, int Rows, int Cols, int Options,

          bool Trivial = internal::use_default_move<T, Size, Rows, Cols>::value>

class DenseStorage : public internal::DenseStorage_impl<T, Size, Rows, Cols, Options> {

  using Base = internal::DenseStorage_impl<T, Size, Rows, Cols, Options>;

  

 public:

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr DenseStorage() = default;

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr DenseStorage(const DenseStorage&) = default;

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr DenseStorage(Index size, Index rows, Index cols)

      : Base(size, rows, cols) {}

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr DenseStorage& operator=(const DenseStorage&) = default;

  // if DenseStorage meets the requirements of use_default_move, then use the move construction and move assignment

  // operation defined in DenseStorage_impl, or the compiler-generated version if none is defined

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr DenseStorage(DenseStorage&&) = default;

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr DenseStorage& operator=(DenseStorage&&) = default;

};

template <typename T, int Size, int Rows, int Cols, int Options>

class DenseStorage<T, Size, Rows, Cols, Options, false>

    : public internal::DenseStorage_impl<T, Size, Rows, Cols, Options> {

  using Base = internal::DenseStorage_impl<T, Size, Rows, Cols, Options>;

  

 public:

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr DenseStorage() = default;

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr DenseStorage(const DenseStorage&) = default;

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr DenseStorage(Index size, Index rows, Index cols)

      : Base(size, rows, cols) {}

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr DenseStorage& operator=(const DenseStorage&) = default;

  // if DenseStorage does not meet the requirements of use_default_move, then defer to the copy construction and copy

  // assignment behavior

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr DenseStorage(DenseStorage&& other)

      : DenseStorage(static_cast<const DenseStorage&>(other)) {}

  EIGEN_DEVICE_FUNC EIGEN_STRONG_INLINE constexpr DenseStorage& operator=(DenseStorage&& other) {

    *this = other;

    return *this;

  }

};
```