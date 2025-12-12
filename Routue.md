**总共 52 周，按周拆分，每周目标明确、产出明确、参考资源可访问。**

---

# 🟦 **学习总目标（一年后能做到）**

- 写出 **CPU 高性能内核（SIMD + cache blocking）**
    
- 写出 **GPU Tiled GEMM（CUTLASS 风格）**
    
- 理解 **PyTorch ATen / DispatchKey / TensorImpl**
    
- 理解 **vLLM / Triton / TensorRT / XLA 的核心路径**
    
- 熟悉 **分布式训练基本机制（DDP、ZeRO、pipeline、tensor parallel）**
    
- 能胜任 **AI Infra / HPC / CUDA 内核工程师** 的工作
    

---

# 🟥 **阶段 0：准备阶段（第 1 周）**

---

# **📘 Week 1：环境搭建 + Linux 工具（40h）**

目标：搭好未来一整年开发环境 + 熟悉工具链。

**任务**

- 安装 Linux（Ubuntu 22.04 LTS）
    
- 配置 CUDA / NSight Systems / Nsight Compute
    
- 配置 gdb / perf / strace / flamegraph
    
- 配置 clang, gcc, ninja, CMake
    
- 学会写一个标准的 CMake 项目骨架
    
- 配置 VSCode + clangd
    
- 配置 Python 环境（PyTorch nightly / CUDA Toolkit）
    

**产出**

- 一个完整可用的 dev 环境仓库
    
- 能独立用 perf record & perf report 检查简单程序
    

**资源**

- CUDA Toolkit：docs.nvidia.com/cuda
    
- Perf 教程：brendangregg.com/perf.html
    
- Nsight Systems docs：docs.nvidia.com/nsight-systems
    

---

# 🟧 **阶段 1：系统与性能基础（Week 2–7，共 6 周）**

重点：操作系统基础、Linux 系统调用、threading、perf、CPU 微架构。

---

# **📘 Week 2–4：Linux/UNIX 系统编程（120h）**

书籍：

- **APUE（Advanced Programming in the UNIX Environment）**
    
- **TLPI（The Linux Programming Interface）**
    

学习内容（按周做）


|章节号 (Chapter)|内容 / 模块|推荐理由|
|---|---|---|
|**4–5**|文件 I/O 基础 & 深入 (open/read/write/close/fcntl / pread/pwrite / scatter-gather / nonblocking I/O / large file)|基本 I/O，是所有系统 + 数据加载/存储 + 日志 + 文件交互基础|
|**6**|进程 (process basics)|理解进程模型、虚拟地址空间、命令行／环境／堆栈布局，对任何系统级/服务级都基础|
|**24–28**|进程创建 / exec / fork / clone / process termination / wait / 子进程管理|dataloader、worker pool、多进程服务 / 分布式启动等，都强依赖 fork/exec/execve/clone|
|**29–33**|线程 (pthread)、同步、thread local storage|多线程 I/O、异步任务、并发 kernel、线程池等必须|
|**49–50**|Memory mapping / Virtual memory operations (mmap, mprotect, mlock, madvise, msync, shared memory mapping)|对大模型 / GPU pinned memory / 数据预处理 / zero-copy IO 非常关键|
|**56–60**|Socket API + TCP/UDP 网络基础 + Server 设计|用于 RPC / inference server / distributed training / parameter server 等网络通信|
|**60 (I/O 多路复用 / 服务器设计)**|epoll / select / poll / server model|构建高性能网络服务 (如 LLM inference 服务) 的基础|
|**20–22 / 23**|信号 (signal) + 计时器 / sleep /定时器|对后台任务、异步 I/O、超时、资源管理、定时任务管理重要|
|**43–48 / IPC / Pipe / FIFO / System V / POSIX IPC / Shared Memory / Semaphore / Message Queue / File Locking**|进程间通信 + 共享内存 + 锁机制|对多进程 / 多 worker / IPC / 数据共享 /锁机制有用 (尤其在 HPC、训练/推理系统)|

### Week 2（40h）

- 进程：fork/exec/wait
    
- 文件描述符、open/close/read/write
    
- 目录、权限、stat
    

### Week 3（40h）

- 线程：pthread、锁、条件变量  
    -线程安全、死锁、内存可见性
    
- 进程间通信：pipe/socketpair/shared memory
    

### Week 4（40h）

- mmap/mprotect/文件映射
    
- 虚拟内存机制
    
- signals (SIGALRM、SIGCHLD)
    
- 定时器 timerfd
    

**产出**

- 自己写的 mini-shell（支持 pipeline）
    
- 使用 mmap 写只读数据库加载器
    
- pthread 线程池
    

---

# **📘 Week 5–7：CPU 架构 + 性能分析（120h）**

书籍：

- **CSAPP 第 6 章（Cache）**
    
- **《计算机体系结构：量化研究》Hennessy & Patterson**
    

学习内容

### Week 5

- Cache line、TLB、page walk
    
- prefetch、false sharing
    
- NUMA 架构
    

### Week 6

- perf、VTune、FlameGraph
    
- CPU pipeline / branch predictor
    
- microbenchmark 方法
    

### Week 7

**项目：写 CPU microbenchmark 套件**

- pointer chasing latency
    
- sequential vs random access
    
- L1/L2/L3 miss ratio
    
- branch mispredict
    

**产出**

- custom profiler
    
- cache-friendly vs unfriendly demo
    
- 完整 profiler 报告（你的第一个作品集项目）
    

---

# 🟨 **阶段 2：现代 C++ + 模板元编程（Week 8–12，共 5 周）**

目标：达到能够阅读 Eigen/CUTLASS 的程度。

---

# **📘 Week 8–9：C++ 模板、CRTP、编译期（80h）**

内容：

- templates / partial specialization
    
- SFINAE / Concepts
    
- type traits
    
- constexpr programming
    
- CRTP deep dive（Eigen 的根基）
    

---

# **📘 Week 10：表达式模板（40h）**

内容：

- operator 重载构造 AST
    
- 惰性求值
    
- inline 展开
    
- 临时对象优化
    

项目：  
**mini-eigen（表达式模板）**

---

# **📘 Week 11–12：SIMD + 内存布局（80h）**

内容：

- AoS vs SoA
    
- std::simd
    
- AVX2/AVX512 intrinsic
    
- aligned load/store
    
- fma/unroll
    

项目：

- SIMD 优化 FMADD kernel
    
- SIMD 版 GEMV
    
- SIMD 对比 naive 性能报告
    

---

# 🟩 **阶段 3：数值计算 + HPC（Week 13–20，共 8 周）**

目标：写出一个可用的 GEMM 内核（CPU 版）。

---

# **📘 Week 13–15：数值分析基础（120h）**

材料：

- Trefethen《Numerical Linear Algebra》
    
- Golub《Matrix Computations》
    

内容：

- LU（部分选主元）
    
- QR（Householder）
    
- SVD（概念级）
    
- Condition number
    
- iterative 方法：CG / GMRES
    

项目：

- 用 QR 实现线性回归
    
- 用 CG 解稀疏系统
    

---

# **📘 Week 16–20：BLAS/Eigen 深入（200h）**

学习 Eigen 源码路径（你最关心的部分）：

- DenseStorage
    
- MatrixBase / PlainObjectBase
    
- CwiseBinaryOp
    
- Map / Block / Transpose
    
- Packet module（SIMD）
    

项目：  
**写一个 mini-BLAS GEMM（CPU）**

- pack
    
- block
    
- micro-kernel（SIMD）
    
- macro-kernel
    
- parallel GEMM
    

产出：

- 自己写的 GEMM 达到 OpenBLAS 15%+
    

---

# 🟦 **阶段 4：CUDA + GPU 内核（Week 21–28，共 8 周）**

---

# **📘 Week 21–22：CUDA 入门（80h）**

内容：

- grid/thread
    
- shared memory
    
- memory coalescing
    
- warp divergence
    
- warp shuffle
    

项目：

- reduction（warp 优化）
    
- transpose（coalescing）
    

---

# **📘 Week 23–24：CUDA 高级（80h）**

内容：

- async copy
    
- shared memory pipeline
    
- warpgroup MMA（Hopper/Blackwell）
    
- LDS / smem bank conflict
    
- parallel reduction tree
    

项目：

- softmax kernel（带 warp 优化）
    
- layernorm kernel
    

---

# **📘 Week 25–28：CUTLASS 深入（160h）**

阅读 CUTLASS 源码：

- tile
    
- warp MMA
    
- epilogue
    
- iterator
    
- pipeline
    

项目：  
**GPU GEMM（mini-CUTLASS）**

- 64×64×32 tile
    
- shared memory double buffer
    
- warp-level MMA
    
- pipelined scheduler
    

性能目标：达到 cuBLAS 30%–50%

---

# 🟪 **阶段 5：AI Infra 深入（Week 29–40，共 12 周）**

---

# **📘 Week 29–32：PyTorch ATen（160h）**

阅读：

- TensorImpl
    
- DispatchKey / Op Registration
    
- TensorIterator
    
- LazyTensor
    
- autograd engine 扩展
    

项目：

- 写一个自定义 CUDA 算子的 PyTorch 扩展
    
- 写 tensor iterator kernel
    

---

# **📘 Week 33–36：编译器（MLIR/XLA/TVM）（160h）**

内容：

- IR（SSA、dialect）
    
- fusion / tiling pass
    
- buffer reuse
    
- schedule tree
    
- lowering pass
    

项目：

- 用 MLIR 做一个 Matmul → Tiled Matmul 的 pass
    
- TVM auto-scheduler 的调优实验
    

---

# **📘 Week 37–40：TensorRT + vLLM（160h）**

内容：

- TensorRT graph fusion
    
- quantization-aware optimization
    
- memory planner
    
- paged attention（vLLM 核心）
    

项目：

- 写一个 vLLM 自定义 kernel（像 PagedAttention 简化版）
    
- 优化推理 pipeline
    

---

# 🟥 **阶段 6：分布式训练（Week 41–48，共 8 周）**

---

# **📘 Week 41–44：PyTorch Distributed**

内容：

- DDP
    
- ZeRO 1/2/3
    
- pipeline parallel
    
- tensor parallel
    
- activation checkpointing
    

项目：

- 实现自己的 DDP-lite
    
- 在 2 卡上训练 Llama 部分层
    

---

# **📘 Week 45–48：轻量级训练引擎（LiteTrain）**

你将写一个简化版的训练框架：

功能包括：

- 数据并行
    
- 梯度规约
    
- mixed precision（AMP）
    
- FP8/FP16 kernel
    

这是一个巨大的作品集项目。

---

# 🟦 **阶段 7：面试准备（Week 49–52，共 4 周）**

---

# **📘 Week 49–50：HPC 基础面试**

- GEMM 推导
    
- cache blocking 原理
    
- SIMD 优化点
    
- false sharing
    
- TLB miss / page walk
    

---

# **📘 Week 51：CUDA 面试**

- warp divergence
    
- SM occupancy
    
- memory coalescing
    
- softmax 瓶颈优化
    
- 附带上机题：写 transpose + reduction + matmul 微 kernel
    

---

# **📘 Week 52：System / PyTorch / Compiler 面试**

- DispatchKey path
    
- autograd backward graph
    
- mlir tiling
    
- kernel fusion
    
- vLLM 如何减少 KV cache 交换