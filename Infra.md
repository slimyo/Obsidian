### 1. 计算机系统基础 (The Foundation)

这是所有 Infra 工程师的底层代码逻辑来源，核心是理解**存储层次结构**。

- **核心教材**：**《Computer Systems: A Programmer's Perspective》(CSAPP)**
    
    - **重点章节**：第 6 章（存储器层次结构）和第 9 章（虚拟内存）。
        
    - **学习目标**：理解 L1/L2/L3 Cache 的工作原理、缓存命中（Cache Hit/Miss）以及为什么局部性（Locality）决定了程序的生死。
        
- **推荐课程**：**CMU 15-213: Introduction to Computer Systems**
    
    - **特点**：CSAPP 的配套课程。它的实验（Labs）极具挑战性，特别是 Cache Lab，会让你亲手写一个缓存模拟器。
        
    - **资源**：[CMU 官方课程主页](https://www.cs.cmu.edu/~213/)，B 站有全套汉化视频。
        

---

### 2. 现代计算架构与并行 (Parallel Architecture)

大模型推理本质上是成千上万个线程在同时处理数据。

- **核心教材**：**《Computer Architecture: A Quantitative Approach》**（体系结构量化研究方法）
    
    - **作者**：John Hennessy & David Patterson (图灵奖得主)。
        
    - **重点内容**：第 4 章（向量、GPU 及并行计算）。这本书会告诉你硬件设计的权衡（Trade-offs）。
        
- **推荐课程**：**UC Berkeley CS61C: Great Ideas in Computer Architecture**
    
    - **重点**：关注 **SIMD** (Single Instruction, Multiple Data) 和 **Thread Level Parallelism** 部分。
        
    - **资源**：[CS61C 官方文档/Notes](https://notes.cs61c.org/)。这个课程非常现代化，2024/2025 年的版本已经包含了大量关于硬件如何适配 AI 负载的内容。


### 一、 核心教材配套资源（第4章重点）

你提到的第4章是理解现代GPU架构（如NVIDIA GPU）和并行计算模型的关键。为了辅助阅读这本“圣经”，建议搭配以下资源：

#### 1. 配套视频课程（辅助理解）

- **普林斯顿大学 ELE 475**：由David Wentzlaff教授主讲，课程参考教材正是《Computer Architecture: A Quantitative Approach》。该课程重点覆盖了第2-5章，讲解非常有激情，适合作为视频版教材学习。
    
- **UC Berkeley CS267**：这是一门关于并行计算的课程，对第4章（数据级并行）的讲解非常深入，特别是关于SIMD和GPU架构的部分。
    

#### 2. 学习笔记与习题

- **知乎专栏《顾大壮不壮》**：该专栏有非常详细的第4章读书笔记，逐节解析了向量架构、SIMD指令集和GPU架构的核心概念，并配有大量图示，非常适合预习或复习。
    
- **CSDN博客**：搜索“计算机体系结构课后习题”或“Computer Architecture A Quantitative Approach 习题解答”，可以找到部分章节的习题答案和解析，帮助巩固量化分析能力。
    

### 二、 AI Infra 实战课程推荐（大模型推理视角）

既然你的目标是学习AI Infra，特别是大模型推理（成千上万个线程并行处理），以下课程将帮你把硬件架构知识转化为系统设计能力：

#### 1. 香港科技大学 CSE 系课程

- **课程名称**：Advanced Large-Scale Machine Learning Systems for Foundation Models
    
- **核心内容**：**Nvidia GPU 计算与通信机制**、大模型训练并行策略（数据/流水线/张量并行）、推理部署（低精度压缩、大规模 batching、offloading）。
    
- **推荐理由**：这门课几乎就是“FM/LLM 训练 + 推理的系统基础设施”课程，直接教你如何把算力、网络、内存组织起来服务大模型工作负载。
    

#### 2. 斯坦福大学 CS336

- **课程名称**：Foundations of Large Language Models
    
- **核心内容**：Transformer架构、MoE、**GPU优化**、并行计算、推理优化。
    
- **推荐理由**：2025年课程已公开全部视频，强调“通过建造来理解”，理论与实践并重，非常适合动手实践。
    

#### 3. NVIDIA DLI 课程

- **课程名称**：模型并行 —— 构建和部署大型神经网络
    
- **核心内容**：使用TensorRT-LLM部署多GPU模型、Triton推理服务器、模型缩减技术。
    
- **推荐理由**：这是工业界最直接的实战课程，教你如何将大型模型部署到生产环境，直接对应大模型推理的工程挑战。
    

### 三、 学习路径建议

1. **打基础**：先看《Computer Architecture: A Quantitative Approach》第4章，配合普林斯顿ELE 475课程视频，理解向量、SIMD和GPU的硬件原理。
    
2. **学系统**：学习香港科技大学或斯坦福的AI Infra课程，了解这些硬件原理是如何被软件（如CUDA、PyTorch）利用来构建大模型训练/推理系统的。
    
3. **做实战**：通过NVIDIA DLI课程或相关开源项目（如vLLM、TensorRT-LLM）进行动手实践，将知识转化为解决实际问题的能力。
    

源


---

### 3. GPU 与算力优化 (Hardware for AI)

AI Infra 目前 90% 的工作是在 GPU 上进行的。

- **入门必读**：**[NVIDIA GPU Architecture Whitepapers](https://www.nvidia.com/en-us/geforce/ada-lovelace-architecture/)**
    
    - **建议**：搜寻 **H100/A100 Tensor Core GPU Architecture** 白皮书。
        
    - **关键点**：理解什么是 **Streaming Multiprocessor (SM)**，以及 **Tensor Core** 为什么比传统的 CUDA Core 计算矩阵快几倍。
        
- **深度学习系统理论**：**[Dive into Deep Learning (D2L) - Chapter: Hardware for Deep Learning](https://d2l.ai/chapter_computational-performance/index.html)**
    
    - **重点**：李沐老师的这本书里专门有几章讲“计算性能”，解释了硬件性能如何影响模型设计。
        

---

### 4. 动手实践工具

- **性能监控**：学习使用 `lscpu`, `top`, 以及 NVIDIA 的 `nvidia-smi` 和 `nvtop`。
    
- **微基准测试**：尝试运行 [Stream Benchmark](https://www.cs.virginia.edu/stream/)，测试你电脑的内存真实带宽。
    

---

### 🛠️ 给你的第一阶段行动计划：

1. **第一周**：阅读 CSAPP 第 6 章，搞清楚什么是 **Memory Wall（存储墙）**。
    
2. **第二周**：观看 CS61C 关于 **Parallelism** 的讲座视频。
    
3. **第三周**：下载一份 A100 GPU 白皮书，画出数据从 HBM 显存经过 L2 Cache 最后进入 SM 的路径图。


## 第二阶段：CUDA 编程核心 (The "How")

这一阶段的目标是：不再把 GPU 当作黑盒，而是能用 CUDA C++ 写出高效的并行算子。

### 1. 核心教材与文档

- **《CUDA C Programming Guide》 (官方文档)**：
    
    - **地位**：这是 CUDA 的“圣经”。不要从头读到尾，重点读 **Programming Model** 和 **Hardware Implementation** 章节。
        
    - **必看点**：理解 Thread Hierarchy（线程层级）如何映射到硬件 SM 上。
        
- **《Professional C++ CUDA C Programming》 (中译名：CUDA C编程权威指南)**：
    
    - **特点**：非常适合工程实践。它详细讲解了如何通过 **Shared Memory** 避免 **Bank Conflict**，以及如何使用 **Stream** 实现计算与传输的并行。
        

### 2. 优质视频与博客

- **[谭升的 CUDA 博客](https://www.google.com/search?q=https://cuda.it1995.com/)**：国内最系统、最易懂的 CUDA 入门教程，适合中文学习者。
    
- **[Unit 2: CUDA Programming (from CS149 at Stanford)](https://www.google.com/search?q=http://cs149.stanford.edu/fall23/)**：斯坦福的并行计算课程。它的 Slide（幻灯片）质量极高，能让你一眼看懂 GPU 调度逻辑。
    

### 3. 练习项目

- **手写 GEMM (矩阵乘法)**：不要直接调库。尝试从最简单的 `v1` 版本优化到使用 `Shared Memory` 的 `v2`，再到使用 `Register Tiling` 的 `v3`。对比你的性能与 NVIDIA 官方库 `cuBLAS` 的差距。
    

---

## 第三阶段：AI 推理引擎架构 (The "What")

这一阶段你将学习如何把零散的 CUDA Kernel 串联成一个支持大模型（如 Llama 3）的推理系统。

### 1. 深度学习框架内核

- **[TinyGrad](https://github.com/tinygrad/tinygrad)**：George Hotz (comma.ai 创始人) 开发。代码量极小，非常适合理解一个推理后端（Buffer 管理、算子调度）是如何从零构建的。
    
- **[FasterTransformer](https://github.com/NVIDIA/FasterTransformer) 源码分析**：虽然它已被 TensorRT-LLM 取代，但其 C++ 代码结构（尤其是针对 Transformer 的优化）是学习 AI Infra 的工业标准模板。
    

### 2. 关键技术专题

- **PagedAttention 论文与源码**：阅读 vLLM 的原始论文。这是目前 AI Infra 岗位的**必考题**。
    
    *
    
- **Flash-Attention 深度解析**：阅读 HazyResearch 的博客或论文。理解它是如何利用 **SRAM** 的局部性来规避 HBM 带宽瓶颈的。
    

---

## 第四阶段：性能分析与编译器 (The "Optimization")

这是区分普通工程师与资深 Infra 工程师的分水岭。

### 1. 性能调优工具 (Profiling)

- **NVIDIA Nsight Systems / Compute**：
    
    - **学习重点**：学会看 **Roofline Model**。它会告诉你你的算子瓶颈到底是在“算力”（Compute Bound）还是在“带宽”（Memory Bound）。
        

### 2. 算子库与编译器

- **[CUTLASS (NVIDIA)](https://github.com/NVIDIA/cutlass)**：学习如何使用 C++ 模板（Template）写出高性能的 CUDA 代码。大模型推理引擎（如 LMDeploy）深度使用了 CUTLASS。
    
- **[TVM / MLIR](https://mlir.llvm.org/)**：如果你想走 AI 编译器方向，学习 LLVM 的中缀表示（IR）和 MLIR 是必须的。推荐阅读 **陈天奇的《Machine Learning Compilation》 (MLC.ai)** 在线课程。
    

---

## 🛠️ 进阶学习路线图 (Action Plan)

|**时间周期**|**学习重点**|**核心任务**|
|---|---|---|
|**第 1-2 个月**|**CUDA 基础**|实现高效的 GEMM，性能达到 cuBLAS 的 70%|
|**第 3-4 个月**|**推理引擎机制**|阅读 `llama.cpp` 源码，搞懂 KV Cache 管理和量化逻辑|
|**第 5-6 个月**|**工程调优**|使用 Nsight Profiler 分析一个开源项目的瓶颈并提交 PR|


以下我为你精选的几个具有代表性、代码质量高且适合学习的 C++ 项目，按学习梯度和技术侧重点排序：

---

### 1. [llama.cpp](https://github.com/ggerganov/llama.cpp) —— **入门与跨平台优化的典范**

这是目前全球最活跃的 C++ 推理项目，没有之一。

- **学习重点**：
    
    - **手写算子**：它几乎不依赖第三方库，包含了大量手写的 GGML 算子。
        
    - **量化技术**：学习其各种量化方法（K-Quants），理解如何降低模型显存占用。
        
    - **硬件后端适配**：通过它你可以看到如何同时支持 CUDA, Metal (Apple Silicon), OpenCL, Vulkan 和 CPU (AVX2/AVX-512)。
        
- **建议**：从阅读 `ggml.c` 开始，理解张量（Tensor）在内存中是如何布局和计算的。
    

### 2. [LMDeploy](https://github.com/InternLM/lmdeploy) —— **工程化与高性能推理**

由书生·浦语团队开发，核心推理后端（TurboMind）采用高性能 C++ 编写。

- **学习重点**：
    
    - **TurboMind 推理引擎**：基于 NVIDIA 的 FasterTransformer 改进，学习如何使用 **CUDA Graph** 和 **Flash-Attention** 进行加速。
        
    - **Continuous Batching**：理解如何通过动态批处理大幅提升服务吞吐量。
        
    - **高性能量化**：它在 W4A16（权重量化）上的优化非常出色。
        
- **建议**：关注其 C++ 后端与 Python 接口的封装方式，这是工业界最常见的架构方案。
    

### 3. [MLC LLM](https://github.com/mlc-ai/mlc-llm) —— **AI 编译器的未来**

这是陈天奇团队的作品，基于 Apache TVM 编译器栈。

- **学习重点**：
    
    - **计算图编译**：它不是“手写”所有算子，而是通过编译器自动生成优化代码。
        
    - **TVM Unity**：学习如何通过 Python 定义模型逻辑，自动编译生成高性能的 C++ 执行代码。
        
    - **全平台覆盖**：包括 WebGPU、安卓、iOS 等极其垂直的端侧优化。
        
- **建议**：如果你对**底层编译器**（Compiler）感兴趣，这个项目是必读的。
    

### 4. [vLLM](https://github.com/vllm-project/vllm) (含 C++ 核心) —— **分布式与显存管理的巅峰**

虽然 vLLM 有大量 Python 代码，但其核心 **PagedAttention** 以及与显存分配相关的逻辑是 AI Infra 的面试必考题。

- **学习重点**：
    
    - **PagedAttention**：模仿操作系统分页机制管理的 KV Cache。
        
    - **多卡并行（Tensor Parallelism）**：学习如何使用 NCCL 进行跨 GPU 的张量并行计算。
        
- **建议**：重点研究其 C++ 核心部分以及它是如何通过 PyTorch C++ 扩展进行交互的。
    

---

### 💡 学习路线与职业建议

对于想从事 AI Infra 的准毕业生，建议按照以下优先级掌握技能：

1. **C++ 进阶**：熟悉 C++17/20，深入理解多线程、内存模型和智能指针。
    
2. **并行计算基础**：
    
    - **CUDA 编程**：能手写简单的向量加法或矩阵乘法 Kernel。
        
    - **SIMD**：了解 AVX2/AVX-512 指令集，这对 CPU 推理至关重要。
        
3. **计算机架构**：理解 GPU 的层次化存储（Shared Memory, Register, L1/L2 Cache）以及 HBM 内存带宽瓶颈。
    
4. **模型知识**：熟练掌握 **Transformer** 架构及其变体（GQA, RoPE, MoE 等），知道推理时每一层在干什么。
    

**你可以尝试的一个小挑战**： 尝试给 `llama.cpp` 增加一个小型算子的优化，或者尝试将一个新出的 Transformer 架构模型手动转换并适配到这些引擎中。