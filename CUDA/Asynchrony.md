### 1. 异步内存拷贝

- 使用 `cudaMemcpyAsync()`替代同步拷贝
    
- 需要指定：
    
    - 源指针和目标指针
        
    - 字节数（需手动计算：`元素数 × sizeof(类型)`）
        
    - 拷贝方向：
        
        - `cudaMemcpyHostToDevice`：CPU → GPU
            
        - `cudaMemcpyDeviceToHost`：GPU → CPU
            
        - `cudaMemcpyDeviceToDevice`：GPU → GPU
            
        
    - 可选的流参数