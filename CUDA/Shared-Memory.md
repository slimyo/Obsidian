- SM的共享内存如下
![[shared.png]]


![[L2.png]]


### 关键发现

- 每个SM有自己的**L1缓存/共享内存**
    
- 共享内存是**软件可控制**的快速内存
    
- 速度比全局内存**快1-2个数量级**
    

## 共享内存 vs 全局内存对比

| 特性        | 共享内存        | 全局内存           |
| --------- | ----------- | -------------- |
| **位置**​   | SM内部（L1缓存）  | GPU板载内存        |
| **速度**​   | 极快（~1-2ns）  | 较慢（~100-200ns） |
| **容量**​   | 较小（每SM几十KB） | 较大（GB级别）       |
| **共享范围**​ | 线程块内        | 全GPU可访问        |
| **管理方式**​ | 软件显式控制      | 硬件自动管理         |
| **访问模式**​ | 需要同步        | 不需要同步          |
| **用例**​   | 线程块内数据交换    | 大规模数据存储        |

## 共享内存的基本用法

### 1. 分配共享内存

```cpp
__global__ void kernel() {
    __shared__ int shared[4];  // 声明共享内存数组
    // 每个线程块有自己的shared[]副本
}
```

### 2. 初始化与同步

```cpp
// 每个线程写入自己的数据
shared[threadIdx.x] = threadIdx.x;

// 必须同步：确保所有写入完成
__syncthreads();

// 现在可以安全读取其他线程写入的数据
if (threadIdx.x == 0) {
    printf("shared[0]=%d, shared[1]=%d, ...", 
           shared[0], shared[1]);
}
```

### 3. 共享内存特性

- **块内共享**：同一线程块的所有线程共享相同的共享内存空间
    
- **块间独立**：不同线程块的共享内存是独立的
    
- **手动管理**：需要显式同步（`__syncthreads()`）
    
- **生命周期**：与线程块相同，线程块结束时释放
    

## 同步解决的问题
### **线程执行乱序问题**

GPU线程虽然在同一warp(32线程)内是同步的，但：

- 不同warp之间**执行顺序不确定**
    
- 线程A的写入可能在时钟周期100完成
    
- 线程B的读取可能在时钟周期90就发生了
    

### **内存可见性问题**

```cpp
// 线程0              // 线程1
shared[0] = 1;        int x = shared[0];  // 可能是0，也可能是1！
```

没有同步，**线程1可能读到旧值0**。

### **实际场景示例**

```cpp
// 块内求和：每个线程把自己数据放入共享内存
shared[tid] = data[tid];
// 如果没有同步，线程0开始求和时，其他线程可能还没写完！
if (tid == 0) {
    float sum = 0;
    for (int i = 0; i < blockDim.x; i++) {
        sum += shared[i];  // 可能读取到未初始化的值！
    }
}
```

---

### 如何同步：`__syncthreads()`

#### 基本用法

```cpp
__global__ void kernel() {
    extern __shared__ float shared[];
    int tid = threadIdx.x;
    
    // 阶段1：写入共享内存
    shared[tid] = ...;
    
    __syncthreads();  // 屏障：等待所有线程到达这里
    
    // 阶段2：读取其他线程写入的数据
    float neighbor_data = shared[(tid + 1) % blockDim.x];
    
    // 更多计算...
    
    __syncthreads();  // 可以多次使用
}
```

### 同步时机示意图

```markdown
时间线：
线程0: 写入shared[0] ──────────┐
线程1: 写入shared[1] ──────┐   │
线程2: 写入shared[2] ────┐ │   │
                        ↓ ↓   ↓
                  __syncthreads()  ← 所有线程在这里等待
                        │ │   │
线程0: 读取shared[1] ────┘ │   │
线程1: 读取shared[2] ──────┘   │  
线程2: 读取shared[0] ──────────┘
```

---


## 在直方图应用中的优化机会

### 优化方案

将线程块的私有直方图从**全局内存**移动到**共享内存**：

```cpp
// 优化前：使用全局内存
cuda::std::span<int> block_histogram;  // 在全局内存中

// 优化后：使用共享内存
__shared__ int block_histogram_shared[BIN_COUNT];
```


## 共享内存设计范式：
### 范式1：平铺（Tiling）——最常用

```cpp
// 矩阵乘法示例
__shared__ float tileA[TILE_SIZE][TILE_SIZE];
__shared__ float tileB[TILE_SIZE][TILE_SIZE];

// 1. 每个线程加载一部分数据到共享内存
tileA[threadY][threadX] = A[globalY][globalX];
tileB[threadY][threadX] = B[globalY][globalX];
__syncthreads();

// 2. 所有线程使用共享内存中的完整数据块计算
for (int k = 0; k < TILE_SIZE; k++) {
    sum += tileA[threadY][k] * tileB[k][threadX];
}
__syncthreads();

// 3. 重复下一个数据块
```

**特点**：

- 块大小通常16×16、32×32等
    
- 适合矩阵乘、卷积等需要重复访问数据的场景
    

---

### 范式2：归约（Reduction）

```cpp
// 求和归约示例
__shared__ float shared[256];
int tid = threadIdx.x;

// 1. 加载到共享内存
shared[tid] = input[globalIdx];
__syncthreads();

// 2. 迭代归约
for (int s = blockDim.x/2; s > 0; s >>= 1) {
    if (tid < s) {
        shared[tid] += shared[tid + s];
    }
    __syncthreads();  // 每次迭代后同步
}

// 3. 线程0写入结果
if (tid == 0) output[blockIdx.x] = shared[0];
```

**特点**：

- 每次迭代后数据减半
    
- 需要多次同步
    
- 适合求和、求最大/最小值等
    

---

### 范式3：滑动窗口

```cpp
// 池化/卷积示例
__shared__ float shared[TPB + 2];  // 额外空间存放边缘数据
int tid = threadIdx.x;

// 1. 加载主数据
if (globalIdx < n) {
    shared[tid + 1] = input[globalIdx];  // 留出边界位置
}
__syncthreads();

// 2. 处理边界（首尾线程加载额外数据）
if (tid == 0) {
    shared[0] = (globalIdx-1 >= 0) ? input[globalIdx-1] : 0;
}
if (tid == blockDim.x-1 && globalIdx+1 < n) {
    shared[blockDim.x+1] = input[globalIdx+1];
}
__syncthreads();

// 3. 计算（每个线程访问shared[tid], shared[tid+1], shared[tid+2]）
output[globalIdx] = shared[tid] + shared[tid+1] + shared[tid+2];
```

**特点**：

- 需要额外空间存放"halo"（光晕）区域
    
- 边界线程负责加载额外数据
    
- 适合卷积、池化、滤波器等
    

---

### 范式4：转置

```cpp
// 矩阵转置示例
__shared__ float tile[TILE][TILE];
int x = blockIdx.x * TILE + threadIdx.x;
int y = blockIdx.y * TILE + threadIdx.y;

// 1. 按行主序加载
tile[threadIdx.y][threadIdx.x] = input[y * width + x];
__syncthreads();

// 2. 按列主序写出（转置）
int x_out = blockIdx.y * TILE + threadIdx.x;
int y_out = blockIdx.x * TILE + threadIdx.y;
output[y_out * height + x_out] = tile[threadIdx.x][threadIdx.y];
```

**特点**：

- 改变数据访问模式
    
- 提高内存访问效率


## 大小问题
### 方法1：编译时常量（最常用）

```cpp
#define SHARED_SIZE 256  // 或用 constexpr

__global__ void kernel() {
    __shared__ float shares[SHARED_SIZE];  // ✅
    // 或
    __shared__ float shares[256];          // ✅
}
```

### 方法2：动态共享内存（运行时指定大小）

```cpp
// 1. 内核声明时不指定大小
extern __shared__ float shares[];  // 不完整类型

__global__ void kernel(int n) {
    int tid = threadIdx.x;
    if (tid < n) {
        shares[tid] = ...;  // 直接使用
    }
}

// 2. 启动时指定大小
int shared_mem_size = n * sizeof(float);
kernel<<<grid, block, shared_mem_size>>>(n);
```

### 方法3：模板参数

```cpp
template<int SHARED_SIZE>
__global__ void kernel() {
    __shared__ float shares[SHARED_SIZE];  // ✅
}

// 调用
kernel<256><<<grid, block>>>();
```

---

## 三种方法对比

|方法|优点|缺点|适用场景|
|---|---|---|---|
|编译时常量|简单，性能好|大小固定|大多数情况|
|动态共享内存|灵活，运行时决定|需要额外参数|大小变化或未知|
|模板参数|类型安全，编译优化|代码膨胀|性能关键，大小有限|