
## 快速的复制方法：
```cpp
// std::copy is much slower than memcpy, so let's introduce a smart_copy which

// use memcpy on trivial types, i.e., on types that does not require an initialization ctor.

template <typename T, bool UseMemcpy>

struct smart_copy_helper;

  

template <typename T>

EIGEN_DEVICE_FUNC void smart_copy(const T* start, const T* end, T* target) {

  smart_copy_helper<T, !NumTraits<T>::RequireInitialization>::run(start, end, target);

}

  

template <typename T>

struct smart_copy_helper<T, true> {

  EIGEN_DEVICE_FUNC static inline void run(const T* start, const T* end, T* target) {

    std::intptr_t size = std::intptr_t(end) - std::intptr_t(start);

    if (size == 0) return;

    eigen_internal_assert(start != 0 && end != 0 && target != 0);

    EIGEN_USING_STD(memcpy)

    memcpy(target, start, size);

  }

};

  

template <typename T>

struct smart_copy_helper<T, false> {

  EIGEN_DEVICE_FUNC static inline void run(const T* start, const T* end, T* target) { std::copy(start, end, target); }

};

  

// intelligent memmove. falls back to std::memmove for POD types, uses std::copy otherwise.

template <typename T, bool UseMemmove>

struct smart_memmove_helper;

  

template <typename T>

void smart_memmove(const T* start, const T* end, T* target) {

  smart_memmove_helper<T, !NumTraits<T>::RequireInitialization>::run(start, end, target);

}

  

template <typename T>

struct smart_memmove_helper<T, true> {

  static inline void run(const T* start, const T* end, T* target) {

    std::intptr_t size = std::intptr_t(end) - std::intptr_t(start);

    if (size == 0) return;

    eigen_internal_assert(start != 0 && end != 0 && target != 0);

    std::memmove(target, start, size);

  }

};

  

template <typename T>

struct smart_memmove_helper<T, false> {

  static inline void run(const T* start, const T* end, T* target) {

    if (std::uintptr_t(target) < std::uintptr_t(start)) {

      std::copy(start, end, target);

    } else {

      std::ptrdiff_t count = (std::ptrdiff_t(end) - std::ptrdiff_t(start)) / sizeof(T);

      std::copy_backward(start, end, target + count);

    }

  }

};

  

template <typename T>

EIGEN_DEVICE_FUNC T* smart_move(T* start, T* end, T* target) {

  return std::move(start, end, target);

}
```