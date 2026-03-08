# 一、入门级（系统 + 性能意识）

## 项目1：实现一个 Mini Deep Learning Framework

参考：

- PyTorch
    

目标：

实现一个 **1000行以内的深度学习框架**。

模块：

Tensor  
Autograd  
Linear layer  
SGD optimizer  
DataLoader

支持：

MLP  
CNN

推荐功能：

automatic differentiation  
computation graph  
backward()

简历写法：

> Implemented a **mini deep learning framework with autograd in 1k LOC**, supporting tensor operations and neural network training.

难度：⭐

---

# 二、GPU与性能

## 项目2：CUDA实现深度学习算子

实现：

vector add  
matrix multiplication  
softmax  
layernorm

优化：

shared memory  
memory coalescing  
warp reduction

工具：

- CUDA Toolkit
    

性能对比：

naive kernel  
optimized kernel  
PyTorch baseline

简历亮点：

> Optimized CUDA kernels achieving **3× speedup over naive implementation**.

难度：⭐⭐

---

# 三、Cache优化（CSAPP经典能力）

## 项目3：实现一个 Cache-friendly GEMM

目标：

实现：

matrix multiply

优化：

loop tiling  
cache blocking  
vectorization

参考：

- Computer Systems: A Programmer's Perspective
    

对比：

naive GEMM  
cache optimized GEMM

性能提升通常：

10x

难度：⭐⭐

---

# 四、PyTorch框架开发

## 项目4：写一个 PyTorch CUDA Operator

基于：

- PyTorch
    

实现：

custom CUDA operator

例如：

fused softmax  
fused layernorm

流程：

CUDA kernel  
C++ extension  
Python binding

最终调用：

import my_operator  
y = my_operator.softmax(x)

难度：⭐⭐⭐

---

# 五、AI编译器

## 项目5：实现一个 Tiny Tensor Compiler

参考：

- TVM
    
- Triton
    

实现：

tensor IR  
operator fusion  
code generation

支持：

add  
matmul  
relu

输出：

CUDA kernel

简历写法：

> Built a **mini tensor compiler with operator fusion and CUDA code generation**.

难度：⭐⭐⭐⭐

---

# 六、LLM推理系统

## 项目6：实现 LLM KV Cache

目标：

实现：

KV cache  
incremental decoding

参考：

- vLLM
    

优化：

paged attention  
memory reuse

效果：

reduce memory usage  
improve throughput

难度：⭐⭐⭐

---

# 七、LLM推理调度

## 项目7：实现 Continuous Batching

参考：

- vLLM
    

实现：

dynamic batching  
request scheduling

效果：

increase GPU utilization

这是很多 **LLM serving系统核心技术**。

难度：⭐⭐⭐⭐

---

# 八、分布式训练

## 项目8：实现 Data Parallel Training

参考：

- DeepSpeed
    
- Horovod
    

实现：

gradient synchronization  
all-reduce  
multi-GPU training

使用：

NCCL

目标：

训练：

ResNet  
Transformer

难度：⭐⭐⭐⭐

---

# 九、最推荐做的 4 个项目（强烈建议）

如果时间有限，我最推荐：

1 CUDA kernel optimization  
2 PyTorch custom operator  
3 LLM KV cache  
4 Continuous batching

因为这是目前 **AI Infra最热门技能组合**。

---

# 十、最强简历组合（2027秋招）

理想项目组合：

CUDA kernel optimization  
PyTorch custom operator  
LLM inference engine  
Tiny tensor compiler

基本对应岗位：

ML Systems Engineer  
AI Infra Engineer  
Inference Engineer  
GPU Engineer

---

# 十一、一个非常现实的建议

很多学生的问题是：

会训练模型  
不会优化系统

但企业真正缺的是：

会让模型跑得更快的人

比如：

- GPU kernel
    
- 推理优化
    
- 编译器
    

这些岗位 **比算法岗竞争小很多**。