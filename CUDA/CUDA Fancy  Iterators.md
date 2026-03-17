### **归约**

- **归约**：`thrust::reduce`
```cpp
// 计算乘积
float product = thrust::reduce(vec.begin(), vec.end(), 1.0f, thrust::multiplies<float>());

// 计算最大值
float max_val = thrust::reduce(vec.begin(), vec.end(), 
                               std::numeric_limits<float>::lowest(),
                               thrust::maximum<float>());

// 计算最小值  
float min_val = thrust::reduce(vec.begin(), vec.end(),
                               std::numeric_limits<float>::max(),
                               thrust::minimum<float>());
```

- `thrust::reduce(thrust::device, vec.begin(), vec.end())`
```cpp
// thrust::reduce 内部执行：
----伪代码
float sum = 0.0f;  // 默认初始值
for (int i = 0; i < vec.size(); i++) {
    sum = sum + vec[i];  // 默认二元操作是加法
}
// 返回：所有元素的总和
----实际GPU上位并行计算，block共享内存sum
真实：仅块结果写入全局内存（N/blockDim.x次）
```

### 转换

- `thrust::make_transform_iterator`
```cpp
auto squared_differences = thrust::make_transform_iterator(
    x.begin(), [mean] __host__ __device__(float value) {
      return (value - mean) * (value - mean);
    });
```