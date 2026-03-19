### CUDA 流

- **定义**：GPU 上按序执行的工作队列
    
- **默认流**：未指定流时的隐式工作队列
    
- **重要特性**：
    
    - 同一流内操作**同步、按序**执行
        
    - 不同流间操作可**异步、并发**执行
        
    
- **流管理函数**：
    
    ```cpp
    cudaStreamCreate(&stream);      // 创建流
    cudaStreamSynchronize(stream);  // 同步特定流
    cudaStreamDestroy(stream);       // 销毁流
    ```
    

### 3. 性能优化策略

#### 初始问题

- 所有异步操作默认在同一流中按序执行
    
- 即使使用 `cudaMemcpyAsync`，后续计算仍需等待拷贝完成
    

#### 解决方案

1. **使用不同流分离计算与拷贝**
    
    - 计算流：执行 GPU 计算
        
    - 拷贝流：执行 CPU-GPU 数据传输


#### 关于`cudaMemcpyAsync`

`cudaMemcpyAsync` actually has an additional parameter that allows you to specify a stream in which the copy operation should be executed:

```c++

cudaError_t

cudaMemcpyAsync(

  void*           dst,

  const void*     src,

  size_t        count,

  cudaMemcpyKind kind,

  cudaStream_t stream = 0 // <-

);

```

- CUB also allows you to specify which stream to use.

```c++

cudaError_t

cub::DeviceTransform::Transform(

  IteratorIn    input,

  IteratorOut  output,

  int       num_items,

  TransformOp       op,

  cudaStream_t stream = 0 // <-

);

```