## 1. **CUDA 开发需要安装的东西**

### **核心组件**（必须安装）

```bash
# 检查你的系统是否已安装
nvcc --version        # CUDA 编译器
nvidia-smi            # GPU 驱动和状态查看
```

### **完整安装清单**：

1. **NVIDIA 显卡**（支持 CUDA 的 GPU）
    
2. **NVIDIA 显卡驱动**（版本要匹配 CUDA）
    
3. **CUDA Toolkit**（包含 nvcc 编译器、CUDA 库等）
    
4. **cuDNN**（用于深度学习的加速库，可选但常用）
    

## 2. **CMake 中配置 CUDA 的完整示例**

### **项目结构**

```markdown
cuda_project/
├── CMakeLists.txt
├── src/
│   ├── cuda/
│   │   ├── kernel.cu
│   │   ├── kernel.h
│   │   └── CMakeLists.txt
│   └── main.cpp
└── CMakePresets.json
```

### **根 CMakeLists.txt**

```cmake
cmake_minimum_required(VERSION 3.18)  # 需要 3.18+ 对 CUDA 支持更好
project(CudaProject LANGUAGES CXX CUDA)  # 关键：声明 CUDA 语言

# 设置 C++ 标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 设置 CUDA 标准
set(CMAKE_CUDA_STANDARD 17)  # CUDA 11+ 支持 C++17
set(CMAKE_CUDA_STANDARD_REQUIRED ON)

# 设置编译选项
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -G -O0")  # -G 生成调试信息
else()
    set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -O3")
endif()

# 启用可分离编译（大型项目需要）
set(CMAKE_CUDA_SEPARABLE_COMPILATION ON)

# 添加子目录
add_subdirectory(src/cuda)
```

### **src/cuda/CMakeLists.txt**（CUDA 模块）

```cmake
# 方法1：创建 CUDA 库
enable_language(CUDA)

# 设置 CUDA 源文件
set(CUDA_SOURCES
    kernel.cu
    kernel.h
)

# 创建 CUDA 库
add_library(cuda_kernels STATIC ${CUDA_SOURCES})

# 设置包含路径
target_include_directories(cuda_kernels
    PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_SOURCE_DIR}/include
)

# 设置 CUDA 架构（必须！）
set_target_properties(cuda_kernels PROPERTIES
    CUDA_ARCHITECTURES "75"  # RTX 2060/2070/2080 是 75
    # 通用写法：native 自动检测
    # CUDA_ARCHITECTURES "native"
    # 或多架构：50;60;70;75;80;86
)

# 方法2：或者创建可执行文件
# add_executable(cuda_test kernel.cu)
# set_target_properties(cuda_test PROPERTIES
#     CUDA_ARCHITECTURES "75"
# )
```

### **kernel.cu 示例**

```cpp
// kernel.cu
#include <iostream>
#include <cuda_runtime.h>

__global__ void addVectors(float* a, float* b, float* c, int n) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < n) {
        c[idx] = a[idx] + b[idx];
    }
}

extern "C" void runCudaKernel(float* h_a, float* h_b, float* h_c, int n) {
    float *d_a, *d_b, *d_c;
    
    // 分配设备内存
    cudaMalloc(&d_a, n * sizeof(float));
    cudaMalloc(&d_b, n * sizeof(float));
    cudaMalloc(&d_c, n * sizeof(float));
    
    // 拷贝数据到设备
    cudaMemcpy(d_a, h_a, n * sizeof(float), cudaMemcpyHostToDevice);
    cudaMemcpy(d_b, h_b, n * sizeof(float), cudaMemcpyHostToDevice);
    
    // 调用核函数
    int blockSize = 256;
    int gridSize = (n + blockSize - 1) / blockSize;
    addVectors<<<gridSize, blockSize>>>(d_a, d_b, d_c, n);
    
    // 拷贝结果回主机
    cudaMemcpy(h_c, d_c, n * sizeof(float), cudaMemcpyDeviceToHost);
    
    // 清理
    cudaFree(d_a);
    cudaFree(d_b);
    cudaFree(d_c);
}
```

## 3. **检查 CUDA 是否安装正确**

```bash
# 1. 检查 CUDA 版本
nvcc --version
# 输出示例：Cuda compilation tools, release 11.8, V11.8.89

# 2. 检查 GPU
nvidia-smi
# 查看 GPU 型号和驱动版本

# 3. 检查 CUDA 示例是否工作
cd /usr/local/cuda/samples
sudo make
./1_Utilities/deviceQuery/deviceQuery
```

## 4. **如果 CMake 找不到 CUDA**

```cmake
# 在 CMakeLists.txt 开头强制查找
find_package(CUDA REQUIRED)
if(CUDA_FOUND)
    message(STATUS "CUDA 版本: ${CUDA_VERSION}")
    message(STATUS "CUDA 路径: ${CUDA_TOOLKIT_ROOT_DIR}")
else()
    message(FATAL_ERROR "CUDA 未找到！请安装 CUDA Toolkit")
endif()

# 手动指定路径
# set(CUDA_TOOLKIT_ROOT_DIR "/usr/local/cuda-11.8")
```

## 5. **编译命令**

```bash
# 使用 CMake
mkdir build && cd build
cmake .. -DCMAKE_CUDA_ARCHITECTURES=75
make

# 或者直接使用 nvcc
nvcc -o my_program kernel.cu -arch=sm_75
```

## 6. **验证代码**

创建简单的测试文件 `test.cu`：

```cpp
#include <stdio.h>

__global__ void helloFromGPU() {
    printf("Hello from GPU thread %d!\n", threadIdx.x);
}

int main() {
    printf("Hello from CPU\n");
    helloFromGPU<<<1, 5>>>();
    cudaDeviceSynchronize();
    return 0;
}
```

编译运行：

```bash
nvcc test.cu -o test
./test
# 应该输出：
# Hello from CPU
# Hello from GPU thread 0!
# Hello from GPU thread 1!
# ...
```

## 7. **常见问题和解决方案**

|问题|解决方案|
|---|---|
|`nvcc: command not found`|安装 CUDA Toolkit，并添加 `/usr/local/cuda/bin`到 PATH|
|`cannot find -lcudart`|添加 `/usr/local/cuda/lib64`到 LD_LIBRARY_PATH|
|`no kernel image is available`|设置正确的 `-arch=sm_XX`匹配你的 GPU|
|CMake 找不到 CUDA|设置 `-DCUDA_TOOLKIT_ROOT_DIR=/path/to/cuda`|

## 8. **GPU 架构对应表**

|GPU 型号|架构代号|-arch 参数|
|---|---|---|
|RTX 4090, 4080|Ada Lovelace|sm_89|
|RTX 3090, 3080|Ampere|sm_86|
|RTX 2080, 2070|Turing|sm_75|
|GTX 1080, 1070|Pascal|sm_61|
|GTX 980, 970|Maxwell|sm_52|

**总结**：你需要：

1. 安装 **CUDA Toolkit**
    
2. 在 CMake 中声明 `LANGUAGES CXX CUDA`
    
3. 设置正确的 `CUDA_ARCHITECTURES`
    
4. 用 `nvcc`编译 `.cu`文件，而不是 `g++/clang++`
    

没有这些配置，`__global__`只是个无法识别的关键字。