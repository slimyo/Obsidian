# 一、AI Infra 的能力结构

AI Infra 的知识结构大致是：

ML模型  
   ↓  
深度学习框架 (PyTorch / JAX)  
   ↓  
计算图 & 编译器  
   ↓  
GPU / CUDA / Kernel  
   ↓  
CPU / Memory / Cache  
   ↓  
OS / Distributed System

所以学习顺序应该是：

计算机系统 → GPU → ML系统 → 分布式训练

---

# 二、阶段1：计算机系统基础（1–2个月）

核心目标：

> **理解程序在机器上的运行**

### 1 系统基础

必读：

- Computer Systems: A Programmer's Perspective
    

重点章节：

3 Machine-Level Representation  
5 Performance Optimization  
6 Memory Hierarchy  
9 Virtual Memory

能力：

- assembly
    
- cache
    
- memory
    
- CPU性能
    

这些直接决定 **深度学习算子性能**。

---

### 2 操作系统

推荐：

- Operating Systems: Three Easy Pieces
    

重点：

process  
threads  
virtual memory  
concurrency

你已经读过  
**The Linux Programming Interface**

所以 **OSTEP 可以很快读完**。

---

# 三、阶段2：GPU 与并行计算（1–2个月）

AI Infra最关键能力之一：

> **理解GPU如何执行深度学习**

推荐书：

- Programming Massively Parallel Processors
    

重点：

SIMT  
warp  
shared memory  
global memory  
coalesced memory access

能力：

- CUDA kernel
    
- GPU memory
    
- parallel algorithm
    

---

### CUDA官方教程

必须掌握：

- CUDA Toolkit
    

学习：

CUDA kernel  
thread block  
warp  
shared memory

实践：

写一个：

vector add  
matrix multiply  
softmax

---

# 四、阶段3：ML系统（核心阶段）

这一步开始接触 **AI Infra核心内容**。

推荐课程：

### Berkeley ML Systems

- CS 294-196: Machine Learning Systems
    

内容包括：

ML pipeline  
data pipeline  
training system  
distributed training  
model serving

---

推荐书：

- Machine Learning Systems: Design and Implementation
    

重点：

training pipeline  
feature pipeline  
serving  
monitoring

---

# 五、阶段4：深度学习框架

要理解：

> PyTorch / TensorFlow 内部如何执行

推荐：

研究：

- PyTorch
    

重点模块：

autograd  
tensor  
dispatcher  
CUDA kernel

源码入口：

aten/  
c10/  
torch/csrc/

建议实践：

写一个：

custom CUDA operator

---

# 六、阶段5：AI编译器

这是未来 **AI Infra最大风口之一**。

需要学习：

### 编译器基础

推荐：

- Engineering a Compiler
    

理解：

IR  
optimization  
code generation

---

### AI编译器

重点项目：

- TVM
    
- XLA
    
- Triton
    

理解：

tensor IR  
operator fusion  
kernel generation

---

# 七、阶段6：分布式训练

当模型变大后，需要：

> **多GPU / 多节点训练**

推荐论文：

- Megatron-LM: Training Multi-Billion Parameter Language Models Using Model Parallelism
    

理解：

data parallel  
tensor parallel  
pipeline parallel

---

推荐系统：

- DeepSpeed
    
- Horovod
    

---

# 八、阶段7：LLM 推理系统（当前最大风口）

当前 AI Infra 最热门领域：

LLM inference optimization

重点系统：

- vLLM
    
- TensorRT
    
- FasterTransformer
    

核心技术：

KV cache  
paged attention  
continuous batching  
quantization

---

# 九、完整学习路线图（推荐）

阶段1 计算机系统  
   CSAPP  
   OSTEP  
  
阶段2 GPU  
   CUDA  
   Programming Massively Parallel Processors  
  
阶段3 ML Systems  
   Berkeley ML Systems  
   ML Systems book  
  
阶段4 Framework  
   PyTorch源码  
  
阶段5 AI Compiler  
   TVM  
   Triton  
   XLA  
  
阶段6 Distributed Training  
   Megatron  
   DeepSpeed  
  
阶段7 LLM Inference  
   vLLM  
   TensorRT

---

# 十、未来2年 AI Infra 最稀缺的方向

结合目前产业趋势（OpenAI / Anthropic / NVIDIA招聘）：

最稀缺的是：

### 1 GPU Kernel Engineer

技能：

CUDA  
kernel optimization  
tensor core

---

### 2 AI Compiler

技能：

IR  
graph optimization  
kernel generation

---

### 3 LLM Inference

技能：

KV cache  
serving  
batch scheduling

---

# 十一、给你的现实建议（非常重要）

如果你是：

> **AI硕士，准备 2027 秋招**

最好的路线是：

CSAPP  
   ↓  
CUDA  
   ↓  
PyTorch源码  
   ↓  
Triton  
   ↓  
vLLM

因为这是：

> **目前AI Infra岗位最多的一条路径**