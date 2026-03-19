
[更简单的CUDA入门（更新版） |NVIDIA 技术博客](https://developer.nvidia.com/blog/even-easier-introduction-cuda)

- github教程：
[accelerated-computing-hub/tutorials/cuda-cpp/notebooks/01.01-Introduction/01.01.01-CUDA-Made-Easy.ipynb at main · NVIDIA/accelerated-computing-hub](https://github.com/NVIDIA/accelerated-computing-hub/blob/main/tutorials/cuda-cpp/notebooks/01.01-Introduction/01.01.01-CUDA-Made-Easy.ipynb)

### 流程指令：

> 构建项目:[[Ninja&Make]]
```python
cmake -G Ninja -B build/debug -DCMAKE_BUILD_TYPE=Debug

ninja -C build/debug
```

> Nsight:[[Nsight&NVIDIA Tools Extension（NVTX）]]
```python
nsys profile -o my_report ./cuda_test

nsys-ui my_report.nsys-rep
```